# LFV 实施设计

> 本文是 `spec.zh-cn.md`（下称 spec，规范）的实施设计：回答"怎么建"——存储结构、内部机制、工程结构。行为与约束以 spec 为准。
> 本文引用 `spec §x` 指规范章节。

## 0. 总览

一句话：**真理源在文件系统（`objects/` + `snapshots.log` + 各 yaml），`index.db` 只是缓存；所有命令 = 惰性扫描 → 解析参数 → 一个 ops 流程 → 渲染。**

数据分四层，依赖方向自上而下：

| 层 | 内容 | 可变性 | 模块 |
| --- | --- | --- | --- |
| Object | File Object / Tree Object | 写一次永不改 | `object` |
| Snapshot | File Snapshot / Tree Snapshot（JSON Lines） | append-only | `snapshot` |
| Refs（用户意志） | `HEAD`、`branches.yaml`、`tags.yaml`、`trees/HEAD`、`trees/tags.yaml`、`config.yaml`、`meta.yaml`、`REPLAY.yaml` | 可变，不可重建 | `reftable`、`tracked`、`tree`、`repo` |
| Index | `index.db` | 可变，可重建 | `index` |

## 1. 工程结构

### 1.1 crate 划分：单 crate，lib + bin

- 一个 package `lfv`，`src/lib.rs` 导出全部核心模块，`src/main.rs` 仅做 clap 解析 + 调度 + 错误输出/退出码。
- **不做 workspace**（`lfv-core` / `lfv-cli` 拆分）。理由：v0.x 单作者、唯一消费者是 CLI；拆 workspace 只增加版本与路径维护成本。lib/bin 分离已足以让 `tests/` 既能走 `assert_cmd` 端到端，也能直接调用 ops 层做不启动进程的快速测试。
- 若将来要给 GUI/守护层复用，`lib.rs` 已在，升级为 workspace 是机械操作。

### 1.2 模块清单与职责

```
src/
├── main.rs        入口：解析 → dispatch → 错误格式化/退出码
├── lib.rs         pub mod 声明；不含逻辑
├── cli/           clap derive 命令树；每个子命令一个文件；只做「参数 → ops 调用 → 渲染」
├── ops/           用例层：§4 每个流程一个函数；跨模块事务的唯一发起者
├── repo/          Repository：寻找仓库根、.lfv 布局路径（layout.rs）、config.yaml（config.rs）、进程锁、init/open
├── object/        ObjectStore：hash、压缩策略、分桶写读、枚举/删除（gc）；TreeManifest（Tree Object 编解码）
├── snapshot/      Snapshot 记录、digest 规范形、SnapshotLog（append / 全量读 / 尾部截断）、事件类型推导、祖先遍历
├── reftable/      name → snap_id 的 YAML 表（branches.yaml / tags.yaml / trees/tags.yaml 同一格式）
├── tracked/       files/<ULID>/ 目录的句柄：meta.yaml、HEAD、分支表、标签表、日志；file-id 分配与退役
├── tree/          trees/ 目录的句柄：日志、HEAD、tree: 标签；Tree Object 的构建
├── index/         SQLite：schema、迁移、查询、rebuild
├── scan/          工作树扫描：.lfvignore、增量失效、auto-track / auto-delete、改名自动识别、状态字母推导；工作区访问层（Unicode 规范化、定位、大小写冲突检测）
├── resolve/       CLI 参数解析：<file> <FO-ish> <FS-ish> <TO-ish> <TS-ish> → 强类型目标
├── diff/          两个字节串的 unified diff（文本）/ 元数据 diff（二进制）
├── merge/         三路文本合并、冲突标记、重放引擎、进行中状态（merge/rebase --continue/--abort）
└── util/          路径规范化、时间、ULID、原子写（tmp + rename）、文件系统小工具
```

分支与标签没有独立模块：两者格式同为 `name: snap:<ULID>` YAML，行为只差"标签只增不改"，由 `reftable` 一个泛型表加各 plane 句柄承担。

### 1.3 依赖方向（只允许向下）

```
cli → ops → { scan, resolve, merge, diff }
                 → { tracked, tree, index, object, snapshot, reftable }
                        → repo → util
```

- `ops` 是唯一允许同时触碰 `tracked`、`tree`、`index`、`object` 并决定写入顺序的层（§3.9 崩溃一致性依赖这一点）。
- `scan` 只写 `index` 与 `tracked`（auto-track 要分配 file-id 建目录），不写 snapshot。
- `resolve` 只读。
- `cli` 不得直接调用 `index` / `object`；所有输出数据来自 ops 返回的结构体（为将来 `--json` 留路，也方便测试断言）。

### 1.4 第三方 crate（对照 spec §6 技术选型）

| 用途 | crate | 备注 |
| --- | --- | --- |
| CLI | `clap` v4 derive | — |
| 错误 | `thiserror` + `anyhow` | 库层 `thiserror`，`ops` 以上 `anyhow` |
| hash / 压缩 | `blake3` / `zstd` | — |
| 序列化 | `serde` + `serde_json` + `serde_yaml_ng` | `serde_yaml` 已归档；只用映射/序列/标量子集 |
| SQLite | `rusqlite`（`bundled`） | 免系统依赖，Windows 主平台 |
| diff / 三路合并 | `diffy` | 冲突标记标签由 `merge` 模块对输出做行首替换（spec §4.7.3） |
| 时间 | `jiff` | — |
| ID | `ulid` | — |
| ignore 规则 | `ignore`（ripgrep 的 gitignore 实现） | 见 §4.2 |
| Unicode 规范化 | `unicode-normalization` | 仅用于工作区访问层，见 §4.2.1 |
| 日志 | `tracing` + `tracing-subscriber` | — |
| 测试 | `assert_cmd` + `predicates` + `tempfile` | — |

### 1.5 测试目录

```
tests/
├── cli_init.rs
├── cli_track.rs
├── cli_snap.rs
├── cli_rewind.rs
└── ...
```

- 单元测试就近放在各 mod 的 `#[cfg(test)]` 中；`snapshot::event_kind` 与 `snapshot::loop_check` 必须覆盖 §3.3.1 表的全部组合。
- 集成测试通过 `assert_cmd` 调用可执行文件；需要检查内部状态时直接用 lib 打开 `.lfv`。

### 1.6 构建与发布

- **构建**：`cargo build` / `cargo build --release`。
- **测试**：`cargo test`（包含单元测试与集成测试）。
- **格式化**：`cargo fmt`，CI 中 `cargo fmt -- --check`。
- **静态检查**：`cargo clippy --all-targets -- -D warnings`。
- **目标平台**：
  - Windows x86_64（开发主平台）；
  - Linux x86_64（GNU 与 musl 两套发布产物）；
  - macOS（aarch64 / x86_64）尽力支持，不做主测。
- **发布物**：单一可执行文件 `lfv(.exe)`，通过 GitHub Releases 提供。
- **版本号**：遵循 SemVer，1.0 之前所有公开命令的语义都可能调整，但破坏性变更必须在 CHANGELOG 中显式声明。

## 2. 核心数据模型

### 2.1 标识符（全部 newtype，禁止裸 String 穿模块边界）

| 类型 | 文本形式 | 说明 |
| --- | --- | --- |
| `FileId` | `file:<ULID 26>` | 文本形式（DB、YAML、日志、CLI）带 `file:` 前缀；`files/` 下的目录名只用 `<ULID>`（Windows 文件名不允许 `:`） |
| `SnapId` | `snap:<ULID 26>` | file / tree 两个 plane 共用格式，全局唯一 |
| `ObjectHash` | `blake3:<64 hex>` | 内部 `[u8; 32]`；File Object 与 Tree Object 同类型，**存储层不区分类型** |
| `TreeId` | = `ObjectHash` | 内容 hash，无独立身份 |
| `BranchName` / `TagName` | 完整命名规则见 spec §4.6（不含 `:` 与控制字符、不含 YAML 指示符字符与首尾空白、非空、不以 `/` 开头或结尾、不是命名空间保留字、同一文件内分支名与标签名不重名）；本文 §2.1.1 说明这些约束的由来 | tree 标签内部存 `tree:<name>`，`TreeTag` 单独类型 |
| `RepoPath` | 规范化后相对仓库根（CLI 输入可相对 cwd，或以 `/` 开头表示仓库内绝对路径）、`/` 分隔、UTF-8，满足 Windows 文件名规则（不含 `<>:"\|?*` 与控制字符、分量不以 `.`/空格结尾、非保留名） | spec §4.2 路径规则 |
| `WorktreeRef` | `work:<path>` | 工作区特殊 File Object 的字面量 |

### 2.1.1 元数据字符串的序列化规则（对应 spec §3.5）

写入这些 yaml 文件的字符串按可控程度分三类，处理方式不同：

**用户可控**（`RepoPath`、`config.yaml` 的 `user.name`、snapshot 的 `author`）——序列化时强制双引号，引号内按标准 YAML 双引号转义规则书写，与 digest 规范化序列化（§3.3.2）复用同一套转义实现（只转义 `"`、`\`、控制字符，非 ASCII 原样 UTF-8）。`RepoPath` 已经禁止 `"`/`\`（spec §4.2），落到这条规则时天然不需要真正转义，直接"扫到下一个 `"`"即可；`user.name`/`author` 没有字符限制，需要按此规则完整转义。

**用户部分可控、由 LFV 批准**（`BranchName`、`TagName`）——序列化时不加引号。完整的命名规则由 spec §4.6 规定，本节说明它为什么长成那样：不加引号的代价是创建时的校验必须挡掉一切会引发 YAML 裸标量歧义的写法。

| 规则 | 理由 |
| --- | --- |
| 禁止控制字符与 `:` | 控制字符会破坏按行组织的文件与输出；`:` 是命名空间分隔符 |
| 禁止 `#?,[]{}&*!\|>'"%@` 及反引号本身 | YAML 指示符字符（c-indicator），在裸标量特定位置有特殊语法含义 |
| 禁止 `\` | 若该值将来出现在需要转义的上下文中，保证零转义 |
| 禁止开头或结尾的空白字符 | YAML 裸标量会裁剪首尾空白，否则写入值与读回值不一致（静默损坏，非解析错误） |
| 禁止整体等于 `-`，或匹配 `/^-\s/`（连字符后紧跟空白） | 裸标量开头 `-` 加空白是 YAML 块序列项标记；`-` 出现在其它位置（不接空格、结尾、中间）正常 |

校验放在创建分支名/标签名的命令（`lfv branch-rename`、`lfv tag`，以及 rewind/detour/rebase/revive 隐式创建的保留分支名与 `tree:detour/...` 标签）里，与命名空间保留字检查、分支/标签重名检查同一层，不符合直接拒绝创建。

**LFV 自己控制**（`snap:<ULID>`、`file:<ULID>`、`blake3:<hex>`、RFC 3339 时间戳、`true`/`false`、整数版本号等）——字符集由 LFV 自身定义且已知安全，裸写，不加引号、不转义。

### 2.2 Object 面

**`object::ObjectStore`**

```rust
pub enum Encoding { Raw, Zstd }
impl ObjectStore {
    /// Stream-hash a worktree file and store it under the compression policy.
    pub fn put_path(&self, p: &Path) -> Result<(ObjectHash, u64 /* raw size */)>;
    /// For in-memory producers: tree manifest, merge results.
    pub fn put_bytes(&self, b: &[u8]) -> Result<ObjectHash>;
    pub fn contains(&self, h: &ObjectHash) -> bool;
    pub fn locate(&self, h: &ObjectHash) -> Option<(PathBuf, Encoding)>;
    pub fn open(&self, h: &ObjectHash) -> Result<Box<dyn Read>>;      // decompresses on the fly
    pub fn read(&self, h: &ObjectHash) -> Result<Vec<u8>>;
    pub fn restore_to(&self, h: &ObjectHash, dest: &Path) -> Result<()>; // tmp + rename
    pub fn iter(&self) -> impl Iterator<Item = (ObjectHash, Encoding)>;  // gc / verify
    pub fn remove(&self, h: &ObjectHash) -> Result<()>;                  // gc only
}
```

写入策略见 §3.2。

**`object::TreeManifest`**（Tree Object 的内存形态，格式见 §3.5）

```rust
pub struct TreeManifest(BTreeMap<RepoPath, ObjectHash>);   // BTreeMap 即 UTF-8 字节序
impl TreeManifest {
    pub fn to_bytes(&self) -> Vec<u8>;          // `- "path": blake3:...\n` per entry; empty => zero bytes
    pub fn parse(b: &[u8]) -> Result<Self>;     // line regex `^- "(.+)": (\S+)$`
    pub fn tree_id(&self) -> ObjectHash;        // blake3(to_bytes())
    pub fn diff(&self, other: &Self) -> Vec<TreeEntryChange>;
}
```

### 2.3 Snapshot 记录

单一 wire 结构，两个 plane 共用；plane 特有不变量在构造函数校验：

```rust
pub struct Snapshot {
    pub id: SnapId,
    pub parent: Option<SnapId>,
    pub path: Option<RepoPath>,      // file: Some; tree: None
    pub object: Option<ObjectHash>,  // file: None = delete; tree: always Some
    pub size: Option<u64>,           // file: Some (0 when object is None); tree: None
    pub created_at: Timestamp,       // RFC 3339, UTC, second precision, `Z`
    pub author: String,
    pub message: String,
    pub digest: Digest,
}
pub enum Plane { File, Tree }
```

- 不含 `tags` 字段：标签真理源是 `tags.yaml`，可删除；记录不可变且受 digest 保护，二者不能共存。
- digest 规范形见 §3.3.2；事件类型推导 `snapshot::event_kind(parent: Option<&Snapshot>, cur: &Snapshot) -> EventKind` 见 §3.3.1。

### 2.4 `snapshot::SnapshotLog`（append-only JSON Lines）

```rust
pub struct SnapshotLog { path: PathBuf, plane: Plane, by_id: HashMap<SnapId, Snapshot>, order: Vec<SnapId> }
impl SnapshotLog {
    pub fn load(path, plane) -> Result<Self>;     // full read; tolerant of a torn last line
    pub fn append(&mut self, s: Snapshot) -> Result<()>;   // fsync; refuses if id exists
    pub fn get(&self, id) -> Option<&Snapshot>;
    pub fn ancestors(&self, from: SnapId) -> impl Iterator<Item = &Snapshot>;   // from inclusive, follows parent
    pub fn children(&self, id) -> Vec<&Snapshot>;   // for `log --graph`; built lazily
    pub fn prune(&self, keep: impl Fn(&SnapId) -> bool) -> Result<()>;    // gc --purge only; drops whole lines, never edits one; tmp + rename
}
```

- 每个 file-id 的日志整体读入内存（单文件历史通常几十到几千行）；tree 日志同理。**不在 `index.db` 缓存快照正文**，只缓存定位信息（§3.6 `snap_locator`）。
- 尾部撕裂：末行无 `\n` 或 JSON 解析失败 → 视为未完成写入，加载时忽略；`append` 前若末字节不是 `\n` 则先截掉该残行。**完整但 digest 不符的行不自动处理**，交给 `verify` 报错。
- `parent` 必须已存在于同一日志（append 时校验）。

### 2.5 `tracked`：file-id 目录

```rust
pub struct TrackedFile {
    pub id: FileId,
    pub meta: FileMeta,                          // meta.yaml
    pub head: BranchName,                        // HEAD
    pub branches: RefTable<BranchName>,          // branches.yaml
    pub tags: RefTable<TagName>,                 // tags.yaml
    log: OnceCell<SnapshotLog>,                  // snapshots.log, lazy
}
pub struct FileMeta {
    pub created_at: Timestamp,
    pub initial_path: RepoPath,                  // path at track time; lets an A-state file survive rebuild-index
    pub retired: Option<Retired>,                // set by relink on the src side
}
pub struct Retired { pub at: Timestamp, pub onto: FileId }
```

- `TrackedStore::create(initial_path) -> FileId`：分配 ULID，建目录，写 `meta.yaml`、`HEAD = main`、空 `branches.yaml`（分支在首条快照时才写入，§3.8）、空日志。
- `head_snapshot()`：`branches[head]` → `log.get`；**分支表中没有当前分支** = 文件已跟踪但尚无快照（状态 `A`）。
- 不变量：`branches` 的每个值都在日志中；HEAD 分支名必在 `branches` 中或日志为空。

### 2.6 `tree`：tree plane

```rust
pub struct TreePlane { log: SnapshotLog /* plane = Tree */, head: Option<SnapId>, tags: RefTable<TreeTag> }
```

`trees/HEAD` 为空文件 ↔ `head = None`（§3.7）。无分支。可达性：`trees/HEAD` 与全部 `tree:` 标签是根；`rewind <TS-ish>` 离开无标签且非目标祖先的 HEAD 时自动打 `tree:detour/<anchor-short>/<n>` 标签（spec §4.5.1）。

### 2.7 工作区状态模型（`scan` 与 `index` 共用）

`index.db` 中每个 tracked 行只存 `status ∈ {modified, unmodified}` 与事实字段；**状态字母在渲染时推导**，不落库：

```
fn flag(row, head: Option<&Snapshot>) -> Flag
  head == None                                     -> A        (tracked, no snapshot yet)
  !row.present                                     -> D        (head.object is Some; row kept on purpose)
  row.path != head.path && row.hash == head.object -> R
  row.path != head.path && row.hash != head.object -> R+M
  row.hash != head.object                          -> M
  else                                             -> unmodified
```

`head == None` 时统一渲染为 `A`，不区分"全新文件"与"该路径此前存在过一个已删除 file-id"两种情况——后者只是在 `A` 行下追加一条提示（§4.4），不改变标志位本身。

`untracked` 行单独一张表（§3.6），只登记盘上存在的路径（§4.2）。

不变量：**同一时刻一条路径至多属于一个活跃（未退役、非 untracked）file-id**；`D` 行占用的是它 HEAD 快照的路径，若该路径重新出现文件，扫描视为同一文件回归（`M` 或 unmodified），而非新文件。

### 2.8 参数解析（`resolve`）

spec §4 导语的参数类型解析为：

```rust
pub struct FileRef   { id: FileId }                                                  // <file>
pub struct SnapTarget { file: Option<FileId> /* None = tree plane */, snap: SnapId }  // <FS-ish>
pub enum FileTarget  { Worktree(FileId, RepoPath), Object(FileId, ObjectHash) }      // <FO-ish>
pub enum TreeTarget  { Object(ObjectHash), Snap(SnapId) }                            // <TO-ish> / <TS-ish>
```

裸 token 的消歧顺序（纯语法判定，不查索引）：`snap:` 前缀（snap-id；归属 file / tree plane 由 `snap_locator` 给出）→ `file:` 前缀（file-id）→ `work:` 前缀（工作区字面量）→ `tree:` 前缀（树标签）→ `blake3:` 前缀（tree-id）→ 含 `:` 时在第一个 `:` 处拆为 `<ref>:<file>`，`<file>` 部分再按本规则解析为 file-id 或路径 → 否则视为路径（`a`、`./a`、`../a` 相对 cwd，`/a/b` 为仓库内绝对路径；拒绝操作系统绝对路径与盘符；输入中 `\` 视同 `/`，开头连续多个 `/` 视同一个；规范化为 `RepoPath`，越出仓库报错；查 `tracked.path`，再查 `tracked.head_path`（待落盘的改名），再查历史中最后拥有该路径且未退役的 file-id，多个候选取其 HEAD 快照 `created_at` 最晚者、仍相同则取 snap-id 最大者）。分支名与标签名不能作为裸 token 出现；在已带 `<file>` 参数的命令中，`<FS-ish>` 位置的裸 token 先按 snap-id，再按该文件的分支名，再按标签名解析；同一文件内分支名与标签名不会重名（spec §4.6 在创建时已挡住），因此后两步不会互相冲突。

裸 `snap:*` 的所属 file-id / plane 由 `index.snap_locator` 给出（§3.6）。

`FileTarget::Object` 携带的 `FileId` 只是解析路径上经过的文件上下文（例如供 `lfv diff <FO-ish>` 推断隐式第二参数 `work:<path>` 的路径），不代表该 File Object 归属于这个文件——File Object 是内容寻址的存储单元，不同 file-id、不同分支都可能引用同一个 hash（spec §3.2.1）。`lfv show <FO-ish>`（无 `--content`/`--out`）要列出"引用该 hash 的所有快照"，因此需要遍历全部 `files/*/snapshots.log` 按 `object == 目标 hash` 匹配，而不能只查解析路径上带出的那一个 file-id；这是一个跨全仓库的扫描，目前没有为此建索引，属于低频诊断命令，可接受全量扫描，若后续成为瓶颈再补 `object_hash → snap_id` 的反向索引。

## 3. 存储结构

### 3.1 目录布局

```
A/                                   # 工作目录
├── docs/note.md                     # 被跟踪文件示例
├── photos/2025/sunset.jpg
└── .lfv/
    ├── config.yaml                  # 仓库级配置（严格 YAML），含 format 版本号
    ├── lock                         # 进程级建议锁，空文件
    ├── index.db                     # 可重建的状态/分支/标签/tree_file_refs 缓存（SQLite）
    ├── objects/                     # 内容寻址对象存储（File Object + Tree Object）
    │   ├── tmp/                     # 写入中的临时对象（与两位十六进制分桶名不冲突）
    │   ├── ab/
    │   │   ├── cdef0123...zstd      # zstd 压缩 File Object
    │   │   └── cdef0123...raw       # 原样 File Object（过小/过大/不可压）
    │   ├── 7f/
    │   │   └── a3bc9d12...raw       # Tree Object（清单 YAML，通常较小 → .raw）
    │   └── ...
    ├── files/                       # 每个跟踪文件的元数据
    │   ├── <ULID>/                  # 目录名为 file-id 去掉 `file:` 前缀后的 ULID
    │   │   ├── meta.yaml            # 文件级元数据（创建时间、初始路径、退役标记）
    │   │   ├── HEAD                 # 当前分支名
    │   │   ├── branches.yaml        # 该文件的分支表
    │   │   ├── tags.yaml            # 该文件的标签表
    │   │   ├── snapshots.log        # append-only 的快照记录（JSON Lines）
    │   │   └── REPLAY.yaml          # merge/rebase 进行中状态；仅在操作期间存在
    │   └── ...
    ├── trees/                       # 全局元数据
    │   ├── snapshots.log            # append-only 的全局快照记录（JSON Lines）
    │   ├── tags.yaml                # tree 标签表
    │   └── HEAD                     # 当前 tree-head，存 snap:<ULID> 或为空
    └── logs/                        # CLI 操作日志（可选，便于调试；v0.x 不实现）
```

### 3.2 对象存储

- 内容寻址：对象身份 = `blake3(原始字节)`；分桶路径中的 hash 前缀来自原始内容，与是否压缩无关；
- 分桶：前 2 个十六进制字符作为目录名，避免单目录文件过多；
- 压缩：对原始字节使用 `zstd`（默认 level **3**）压缩后落盘，扩展名 `.zstd`；读对象时在内存中解压；
- 原样存储（扩展名 `.raw`）：满足以下 **任一** 条件时不压缩：
  - **floor**：`size < min_bytes`（默认 **4 KiB**，4096 字节）——过小，帧开销可能使压缩后更大；
  - **ceiling**：`size >= max_bytes`（默认 **16 MiB**，16777216 字节）——避免大文件整段读入内存时的峰值；
  - **无效压缩**：尝试压缩后 `compressed_len >= original_len`（`reject_if_larger`，默认 **true**）——对已压缩二进制等无收益内容；
- 跨文件跨分支共享：相同内容只存一份对象，节省空间。

**写入流程**（`ObjectStore::put_path`）：

- `size < min_bytes` 或 `size >= max_bytes` → 单遍：边读边 hash 边写 `objects/tmp/<random>`，得到 hash 后 rename 到 `objects/ab/<rest>.raw`。大文件不进内存。
- 否则读入内存，hash，zstd 压缩；`compressed_len >= original_len` 且 `reject_if_larger` → `.raw`，否则 `.zstd`。
- 目标已存在 → 直接丢弃临时文件（去重）。同一 hash 只允许一种扩展名存在；`verify` 报告两者并存。
- 存储层不区分 File Object 与 Tree Object；零字节文件与空清单是同一个对象，这是预期行为。

#### 3.2.1 压缩默认配置（`config.yaml`）

```yaml
compression:
  enabled: true
  algorithm: zstd
  level: 3              # zstd 1..=22
  min_bytes: 4096       # floor: size < min → .raw
  max_bytes: 16777216   # ceiling: size >= max → .raw
  reject_if_larger: true
```

### 3.3 快照记录格式

`snapshots.log` 每行一个 JSON 对象（JSON Lines）：

```json
{
  "id": "snap:01HXYZ...",
  "parent": "snap:01HXYY...",
  "path": "docs/note.md",
  "object": "blake3:abcdef0123...",
  "size": 12345,
  "created_at": "2026-05-17T09:21:33Z",
  "author": "goosy",
  "message": "fix typo in title",
  "digest": "blake3:fedcba9876..."
}
```

字段说明：

- `id`：单调可读的快照标识符，采用 [ULID](https://github.com/ulid/spec) 加 `snap:` 前缀。ULID 自带时间戳前缀 + 随机后缀，便于在 `log` 中按时间排序，也便于人在终端粘贴。
- `parent`：父快照 id；分支首个快照为 `null`。
- `path`：**此次快照发生时该文件在工作树上的相对路径**。改名/移动事件就体现为本字段与 `parent.path` 不同；若文件当前不存在，则保留此前盘上最后已知的路径（便于 `log` 阅读）。
- `object`：所引用 Object 的 blake3 hash（带 `blake3:` 前缀以便日后切换算法）；**当文件当前不存在时，此字段为 `null`**。
- `size`：对应 Object 的字节数；`object = null` 时为 `0`。
- `created_at`：UTC、秒级精度、RFC 3339 `Z` 后缀。
- `author`：取 `config.yaml` 的 `user.name`，缺省取操作系统用户名。
- `digest`：**对本条快照记录除 `digest` 字段以外的所有字段做规范化序列化后的 `blake3` 哈希**，用于防篡改校验，规范形见 §3.3.2。`lfv verify` 会逐行重算并比对。
- 记录**不含标签**：标签是外部可变指针（§3.8）。

快照不记录所属分支——分支是外部的可变指针（`branches.yaml`），快照只是历史事实。一条快照可以同时属于多个分支的历史路径，删除某个分支不影响快照本身。

设计为 append-only，便于增量备份和审计。

#### 3.3.1 事件类型的推导

LFV 不在 Snapshot 中显式存"事件类型"字段，而是根据 `(parent, path, object)` 三元组与父快照的差异 **派生**：

| 父 path | 父 object | 当前 path | 当前 object | 事件类型 | `log` 中渲染 |
| ---- | ---- | ---- | ---- | ---- | ---- |
| —（首条） | — | P | O | **add**（首次 snap） | `A  P` |
| P0 | O0 | P0 | O1 | **modify** | `M  P0` |
| P0 | O0 | P1 | O0 | **rename** | `R  P0 -> P1` |
| P0 | O0 | P1 | O1 | **rename + modify** | `R+M  P0 -> P1` |
| P0 | O0 | P0 或 P1 | **null** | **delete** | `D  P0`（P1 ≠ P0 时渲染 `D  P1`，即盘上最后已知路径） |
| P0 | **null** | P1 | O1 | **revive** | `+  P1` |

不会出现的组合，写侧拒绝、读侧报数据完整性错误：

- 首条快照 `object = null`：文件尚无快照即被删除时，`lfv snap` 只撤销跟踪，不产生快照（spec §4.3）。
- 父 object `null`、当前 object `null`：重复删除。删除快照落盘后状态表行已删，snap 不会再处理该文件。
- 父与当前 path、object 全同：spec §4.3 拒绝"未变化"的 snap，且没有 `--allow-empty`。

`revive` 行只会由 `lfv relink` 的接缝产生（§4.7）；`lfv revive` 命令实现为 rewind（§4.10），不追加快照。若 relink 接缝的 O1 在 dst 分支上曾出现于连续 run 之外，同样按环回拒绝（§4.13）。

> `track` 动作只登记 `file-id` 与跟踪标志，**不产生 Snapshot**。Snapshot 链的第一条由用户显式执行 `lfv snap` 产生，事件类型推导为 `add`（`parent=null`，`object≠null`）。`track` 与首次 `snap` 是两个独立步骤，自动 track 亦然。

> 不显式存事件类型的好处：每条 Snapshot 都是完整的"在此刻文件是怎样的"快照，添加新事件类型时无需引入新字段或迁移旧记录；坏处：事件类型必须由读侧推导，所以渲染逻辑要单元测试覆盖全部组合。

> **关于"为什么 Snapshot id 用 ULID 而不是内容 hash"**：
> git 用 commit 的内容 hash 作为 id，天然防篡改但 id 不可预读、不可按时间排序。LFV 选择 ULID 作为 id，因为：
> 1. 单文件场景下，用户经常需要在 `log` 输出里凭肉眼按时间挑选快照；ULID 的可读性远好于纯 hash；
> 2. 防篡改职责交给单独的 `digest` 字段，语义清晰，便于 `verify` 命令针对性校验；
> 3. id 与 digest 解耦，将来想增删元数据字段时不会引起 id 漂移。

#### 3.3.2 digest 规范形

这是跨版本兼容承诺，与磁盘上那一行的空白、字段顺序无关：

1. 取除 `digest` 外的字段，按 §2.3 结构体声明顺序组成 JSON 对象：`id, parent, path, object, size, created_at, author, message`。tree 快照的 `path` 与 `size` 以 `null` 参与，两个 plane 共用一套规范形。
2. 紧凑序列化：无空白；字符串按 `serde_json` 默认规则转义（只转义 `"`、`\`、控制字符，非 ASCII 原样 UTF-8）；`None` → `null`；整数十进制。
3. `digest = "blake3:" + hex(blake3(bytes))`。

`verify` 从解析后的结构体重算，再与记录中的 `digest` 比对。

### 3.4 Tree Snapshot 格式

**Tree Snapshot**（`.lfv/trees/snapshots.log`，每行一个 JSON 对象）：

```json
{
  "id": "snap:01HABC...",
  "parent": "snap:01HABZ...",
  "path": null,
  "object": "blake3:7fa3bc9d...",
  "created_at": "2026-05-17T10:00:00Z",
  "author": "goosy",
  "message": "第三章完成",
  "digest": "blake3:fedcba98..."
}
```

与 File Snapshot 的区别：
- `path` 字段为空（tree 不需要定位路径）。
- `object` 不可为 null（tree 快照始终为对象——"整个工作目录为空"时清单为零字节，blake3 hash 照常计算，是合法的 tree-id）。
- 没有 `size` 字段（规范形中以 `null` 参与）。

### 3.5 Tree Object 格式

**Tree Object**（存放于 `objects/`，内容寻址）采用 YAML 块序列格式，每条目一行：

```yaml
- "docs/ch1.md": blake3:abc123...
- "docs/ch2.md": blake3:def456...
- "docs/ch3.md": blake3:ghi789...
```

#### 3.5.1 格式规范

- **结构**：YAML 块序列（block sequence），每行一个单键映射（single-key mapping）。
- **键**（path）：始终加双引号。路径仅含 Unix 分隔符 `/`；按 spec §4.2 路径规则，可跟踪路径不含 `"`、`\` 与控制字符，因此键内无需任何转义。
- **值**（hash）：`blake3:` 前缀 + 小写十六进制，不加引号（不含任何 YAML 特殊字符）。
- **排序**：条目按 path 字典序（UTF-8 字节序）严格升序排列，不允许重复路径。
- **空清单**：工作目录下无有效跟踪文件时，Tree Object 内容为空字节串（零字节），其 blake3 hash 照常计算，是合法的 tree-id。
- **无多余内容**：不含注释、不含空行、不含 BOM、文件末尾恰好一个换行符（`\n`）；空清单则为零字节，没有换行符。

#### 3.5.2 规范化与 tree-id

blake3 直接对上述字节串摘要，得到该 Tree Object 的身份（tree-id）。由于格式完全确定（引号、排序、换行），同样的工作目录内容必然产生逐字节相同的序列化结果，进而产生相同的 tree-id——无需额外的规范化步骤，存储格式即规范化格式。

两次内容完全相同的工作目录状态——无论何时、以何种方式达到——产生相同的 Tree Object hash，`objects/` 里只有一份条目。

#### 3.5.3 存储与兼容性

Tree Object 通常很小（每个跟踪文件一行），低于 `min_bytes` 压缩下限，以 `.raw` 存储。

该格式是合法的 YAML 子集，外部程序可直接用标准 YAML 解析器读取，无需了解 LFV 内部约定。LFV 自身解析时，也可用一行正则 `/^- "(.+)": (\S+)$/` 逐行提取，不依赖完整解析器。

**对比两个 Tree Snapshot**：逐条比对各自 Tree Object 清单。`path` 相同且 `object` hash 相同——未变；`path` 相同但 `object` hash 不同——已修改；仅出现在其中一个清单里——新增或删除。

### 3.6 可变索引（Mutable Index）与 `index.db` 表结构

可变索引位于 `index.db`，记录当前工作目录中各文件的工作区状态及分支/标签缓存。它是可重建的当前状态缓存，不属于不可变历史对象。存储对象与 Snapshot 链不可篡改，它们与 `.lfv` 下的 yaml / HEAD 文件共同构成仓库的真理源；`index.db` 出错可以随时从真理源重建。

`index.db` 只存放**可重建**的缓存数据，真理源均在文件系统（Snapshot 链、yaml 文件）中。各表说明：

| 表 | 说明 | 可重建 |
| ---- | ---- | ---- |
| `tracked` | 当前拥有路径的跟踪文件（含盘上消失、待 `lfv snap` 记录删除的 `D` 行）。主键 file-id；缓存 HEAD 快照的 path/object/size 与上次扫描的盘上 mtime/size/hash，供增量扫描短路。 | ✓ |
| `untracked` | `config.yaml` 动态 untracked 列表 ∩ LFV 可见 ∩ 盘上存在的路径；保留已知的前 file-id 供 `lfv track` 复用。 | ✓ |
| `scan_meta` | 扫描元数据：`last_completed_at`、`.lfvignore` 与 `config.yaml` 的 mtime，供增量扫描失效判定。 | ✓ |
| `branches` | 每个文件的分支指针缓存。真理源为 `.lfv/files/<ULID>/branches.yaml`。 | ✓ |
| `tags` | 每个文件的标签缓存。真理源为 `.lfv/files/<ULID>/tags.yaml` 及 `.lfv/trees/tags.yaml`。 | ✓ |
| `snap_locator` | 裸 `snap:<ULID>` → 所属 file-id（tree plane 为 NULL）。真理源为各 `snapshots.log`。 | ✓ |
| `tree_file_refs` | tree 维度的反向引用缓存。真理源为 `.lfv/trees/snapshots.log` + `objects/`，`rebuild-index` 时重建；`lfv snap --tree` 时写入。供 `lfv log <FS-ish>` 渲染时附加 tree 关联信息，以及 `lfv rewind <TS-ish>` 把清单条目映射到 file-id。 | ✓ |

> [!note] tracked / untracked 说明
> - `tracked` **正常仅登记 LFV 可见、且工作树上当前存在的文件**（`stat` 成功且不匹配 `.lfvignore` 的路径），`present = 1`；
> - **例外**：已跟踪、盘上消失、待 `lfv snap` 记录当前 FS 状态的条目保留，`present = 0`，`status` 输出时显示为 `D`（§4.5）；
> - `untracked` 行对应 `config.yaml` 动态 untracked 列表且盘上存在的路径；盘上已消失则删除该行（`config.yaml` 列表可保留）；
> - 主键是 file-id 而不是路径：待落盘的改名（`R`）"当前路径 ≠ HEAD 路径"、以及 `D` 行都需要按 file-id 定位。`tracked_path` 部分唯一索引保证盘上存在的文件一路径一行。
> - 对已跟踪且盘上仍存在的文件，`path → file_id` 供 CLI 解析路径；盘上消失后仍可用 `file-id` 定位。

#### 3.6.1 Schema

```sql
-- tracked files that currently own a path (or are pending deletion, present = 0)
CREATE TABLE tracked (
    file_id       TEXT PRIMARY KEY,  -- file:<ULID>
    path          TEXT NOT NULL,     -- current worktree path (= head_path unless a rename is pending)
    present       INTEGER NOT NULL,  -- 1 = exists on disk; 0 = missing (rendered as D)
    status        TEXT NOT NULL,     -- 'modified' | 'unmodified'
    head_snap     TEXT,              -- NULL when no snapshot yet (A)
    head_path     TEXT,
    head_object   TEXT,              -- blake3:...; never NULL when head_snap is set (a deleted HEAD has no row)
    head_size     INTEGER,
    disk_mtime_ns INTEGER,           -- baseline for the mtime/size -> hash short circuit
    disk_size     INTEGER,
    disk_hash     TEXT               -- hash of the on-disk bytes at the last scan
);
CREATE UNIQUE INDEX tracked_path ON tracked(path) WHERE present = 1;

-- dynamically untracked paths that exist on disk (config.yaml untracked list ∩ visible ∩ on disk)
CREATE TABLE untracked (
    path      TEXT PRIMARY KEY,
    file_id   TEXT,                  -- former file-id, if any; reused by `lfv track`
    size      INTEGER NOT NULL
);

-- scan metadata
CREATE TABLE scan_meta (
    key        TEXT PRIMARY KEY,     -- 'last_completed_at' | 'lfvignore_mtime' | 'config_mtime'
    value      TEXT NOT NULL
);

-- branch pointer cache (truth: .lfv/files/<ULID>/branches.yaml)
CREATE TABLE branches (
    file_id    TEXT NOT NULL,        -- file:<ULID>
    name       TEXT NOT NULL,
    snap_id    TEXT NOT NULL,        -- snap:<ULID> at the branch HEAD
    PRIMARY KEY (file_id, name)
);

-- tag cache (truth: .lfv/files/<ULID>/tags.yaml and .lfv/trees/tags.yaml; tree tags use file_id = '')
CREATE TABLE tags (
    file_id    TEXT NOT NULL,
    name       TEXT NOT NULL,        -- tree tags keep their 'tree:' prefix
    snap_id    TEXT NOT NULL,
    PRIMARY KEY (file_id, name)
);

-- where does a bare snap:<ULID> live?  NULL file_id = tree plane
CREATE TABLE snap_locator (
    snap_id    TEXT PRIMARY KEY,
    file_id    TEXT
);

-- tree-side reverse references (truth: trees/snapshots.log + objects/)
CREATE TABLE tree_file_refs (
    tree_snap_id  TEXT NOT NULL,     -- snap:<ULID>
    file_id       TEXT NOT NULL,     -- file:<ULID>
    file_object   TEXT NOT NULL,     -- blake3:...
    PRIMARY KEY (tree_snap_id, file_id)
);
```

**`tree_file_refs` 的写入与重建**：`lfv snap --tree` 创建 Tree Snapshot 时展开 Tree Object 清单，逐 file 插入一行（此时每个条目的 file-id 直接来自 `tracked` 行）；`rebuild-index` 时按 §3.10 的归属规则重建。供 `lfv log <FS-ish>` 渲染时附加 tree 关联信息：

```
snap:01HXYZ  M  docs/note.md   "add chapter 2"
             └─ tree: "第三章完成" [tree:v1.0]
snap:01HWWW  M  docs/note.md   "fix typo"
             └─ tree: "日常归档 2026-05-30"
```

### 3.7 不可重建的状态文件

以下文件记录**用户意志**或操作进行中的状态，无法从 Snapshot 链机械推导，`rebuild-index` 时不覆盖：

**`.lfv/files/<ULID>/HEAD`**

每个跟踪文件一个，内容为当前所在分支名（纯文本，一行）：

```
main
```

LFV 不支持 detached HEAD——`rewind` 强制新建分支保留旧 HEAD，所以 HEAD 始终指向一个具名分支，不会是裸 snap\_id。

**`.lfv/files/<ULID>/meta.yaml`**

```yaml
created_at: 2026-05-17T09:21:33Z
initial_path: "docs/note.md"      # path at track time; the only record of it before the first snapshot
retired: ~                        # ~ until `lfv relink <this> --onto <onto>`, then a nested block:
                                   #   at: 2026-06-01T08:00:00Z
                                   #   onto: file:01HA7BCD...
```

`initial_path` 让尚无快照的文件在 `rebuild-index` 后仍能回到 `tracked` 表而不被再次分配 file-id；`retired` 让退役 file-id 在重建时不与 `onto` 一侧争抢路径。

**`.lfv/files/<ULID>/REPLAY.yaml`**

merge / rebase 进行中的状态（结构见 §4.14），仅在 `--continue` / `--abort` 之间存在。存在该文件时，其它会修改该文件历史的命令拒绝执行，**包括对同一 file-id 重新发起 `lfv merge`/`lfv rebase`/`lfv merge --pick`**：一个 file-id 同时只能有一个进行中的 merge/rebase/pick，必须先 `--continue` 完成或 `--abort` 放弃，才能开始新的一个。

**`.lfv/trees/HEAD`**

tree 层当前头部，内容为 `snap:<ULID>` 或空（仓库尚无 Tree Snapshot 时）：

```
snap:01HABC...
```

**`.lfv/lock`**

进程级建议锁（`std::fs::File::try_lock`，Rust ≥ 1.89）。`Repository::open` 时获取，拿不到即报错退出——两个 `lfv` 进程同时操作同一仓库不受支持。

**`.lfv/config.yaml`**

```yaml
format: 1                 # repository format version; bump on incompatible layout changes
user:
  name: "goosy"
compression: ...          # see §3.2.1 for the full block-style mapping
rename:
  autodetect: true
untracked:                # dynamic untracked list, RepoPath strings
  - "drafts/local.md"
```

`lfv config <key> [value]` 以点分键读写（`user.name`、`rename.autodetect`）。

### 3.8 分支与标签文件（真理源）

以下 yaml 文件是分支/标签的真理源，`index.db` 中的 `branches`、`tags` 表是其可重建缓存。

**`.lfv/files/<ULID>/branches.yaml`**

key = 分支名，value = 该分支当前 HEAD 的 `snap:<ULID>`：

```yaml
main: snap:01HXYZ...
rewind/7RQ2M9KA/1: snap:01HABC...
```

- 分支首次创建时写入；`lfv snap` 在当前分支上产生新快照后更新对应 value。
- `lfv branch-delete` 删除对应 key；快照本身不受影响（append-only）。

**`.lfv/files/<ULID>/tags.yaml`**

key = 标签名，value = `snap:<ULID>`。标签一旦创建不可改指向；删除后同名可重建：

```yaml
v1.0: snap:01HXYZ...
stable: snap:01HWWW...
```

**`.lfv/trees/tags.yaml`**

tree 层标签表，key = `"tree:<name>"`，value = `snap:<ULID>`，标签全局唯一，同样不可改指向、删后可重建：

```yaml
tree:v1.0: snap:01HABC...
tree:release: snap:01HZZZ...
```

用户输入的标签名不允许包含 `:`（用于命名空间隔离）；`tree:` 前缀由 LFV 自动添加。`rewind <TS-ish>` 自动打的 `tree:detour/<anchor-short>/<n>` 也在此表。

所有 yaml / HEAD 写入统一走 `util::atomic_write`（同目录 tmp + rename；Windows 上 `rename` 可覆盖）。

### 3.9 写入顺序与崩溃一致性

一次 `snap` 的写入顺序（`ops::snap_file`）：

1. `ObjectStore::put_*`（幂等，可重复）
2. `SnapshotLog::append`（append + fsync）
3. `branches.yaml` 原子替换（首条快照时同时写入分支）
4. `index` 事务：`tracked`、`branches`、`snap_locator`

任一步之后崩溃的后果：1 后 → 多一个无引用对象（`gc` 回收）；2 后 → 一条悬空快照（`gc --purge` 回收）；3 后 → 索引落后于真理源，由 `lfv rebuild-index` 修复。`ops` 层保证 3→4 在同一函数内完成，且真理源写入永远先于索引写入——索引只会落后，不会领先。

### 3.10 `rebuild-index` 算法

1. 重建 schema。
2. 遍历 `files/*/`：读 meta、HEAD、branches、tags、日志。写 `branches`、`tags`、`snap_locator`。`retired` 的 file-id 不写 `tracked`（但仍写 `snap_locator`，`lfv log file:<src>` 要能用）。按当前分支 HEAD 快照：
   - 有且 `object != null` → `tracked` 行：`path = head.path`，`present = stat 成功`，`head_*` 填充，`disk_*` 置空（迫使下次扫描重新 hash）；
   - 有且 `object == null` → 已删除，无行；
   - 无快照 → `path = meta.initial_path`，`present = stat`，`head_* = NULL`。
3. tree plane：读 `trees/snapshots.log` → `snap_locator(file_id = NULL)`；`trees/tags.yaml` → `tags`；`tree_file_refs` 按下面的归属规则重建。
4. `untracked`：`config.untracked ∩ 可见 ∩ 盘上存在`；`file_id` 取历史中最后拥有该路径且未退役的 file-id（可为空）。
5. 清空 `scan_meta` → 下次扫描为全量。

**`tree_file_refs` 归属规则**：Tree Object 只有 `{path, hash}`，没有 file-id。对每个清单条目 `(P, H)`，在所有 file-id 日志中找 `path == P && object == H && created_at <= tree.created_at` 的快照，取 **created_at 最晚**者所属 file-id。此规则对"删除后同路径新文件"和 relink（复制链的 ULID 更新，时间更晚）两种情况都给出正确答案；残余歧义只在同一时刻两个 file-id 持有相同 `(P, H)`，正常操作不会发生。

### 3.11 可达性与 gc

- 根：file plane = 所有 file-id（含退役）的所有分支头 + 所有标签 + 存在的 `REPLAY.yaml` 里引用的快照；tree plane = `trees/HEAD` + 所有 `tree:` 标签。
- 可达快照 = 从根沿 `parent` 闭包。可达对象 = 可达快照的 `object` ∪ 可达 Tree Snapshot 清单中的条目。
- `lfv gc`：删除**任何快照（含悬空）都不引用**的对象；不动日志。
- `lfv gc --purge`：先按可达性对每个日志执行 `SnapshotLog::prune`：不可达快照整行丢弃，保留的行逐字节不变——这是日志除追加之外唯一的变更，只删除、从不改写已有记录；再按新的引用集删除对象。两步之间崩溃是安全的（只会多留对象）。

## 4. 内部机制

### 4.1 命令执行骨架

```
cli::<cmd>::run(args)
  → repo::Repository::open(cwd)          // find .lfv upward; take lock; load config
  → ops::scan::lazy(&repo, mode)?        // only for commands in §4.3.3
  → resolve::*(&repo, args)              // typed targets (§2.8)
  → ops::<flow>(&mut repo, targets, opts) -> Report
  → cli render(Report)
```

### 4.2 `.lfvignore` 与 `config.yaml`

默认全跟踪会把临时文件、构建产物等也纳入扫描；`.lfvignore` 与 `config.yaml` 提供了不同级别的「排除列表」，它们的职责不同：

| | `.lfvignore` | `config.yaml`（动态 untracked） |
| ---- | ---- | ---- |
| 性质 | 静态、仓库级、**最高优先级** | 用户通过 `lfv untrack` / `lfv track` 维护的动态策略 |
| 状态表 | **不进入** `tracked` / `untracked` | 盘上**存在**时登记 `untracked`；盘上消失则**删除**状态表行（config 列表保留） |
| 工作树扫描 | **不可见**：剪枝跳过，不参与 OS 新建/删除检测 | **可见**：参与 §4.3；仅对盘上存在的路径维护状态表行 |
| `lfv track <file>` | 匹配则**报错**，须改 `.lfvignore` | 从 untracked 列表移除并进入跟踪 |
| `lfv untrack <file>` | 匹配则**报错** | 写入 config；盘上存在则登记 `untracked` |
| `lfv status --include-untracked` | **不会出现** | **唯一来源**（状态表中的 `untracked` 行，均源于 config） |

补充约定：

- `.lfv/` 目录本身永远隐式视为 `.lfvignore` 规则，不进入状态表；
- 匹配 `.lfvignore` 的路径，对 LFV 而言等同于**不存在**：不 auto-track、不 untrack、不列入 `--include-untracked`；
- `rebuild-index` 时，`untracked` 行 = `config.yaml` 动态列表 ∩ LFV 可见路径 ∩ **盘上存在**的路径；
- `.lfvignore` 语法为 gitignore 语法子集，只认仓库根目录的一个文件，用 `ignore` crate 的 gitignore 匹配器；目录模式参与遍历剪枝；
- 动态 untracked 列表按路径记录；untracked 的文件在盘上改名后，新路径是可跟踪候选，会被 auto-track。

#### 4.2.1 工作区访问层（`scan::fs`）

所有按 `RepoPath` 读写工作区的操作都经过这一层（`ops` 在调用 `object::restore_to` 等之前，先由它把 `RepoPath` 解析为磁盘路径）。

- **规范形**：`RepoPath` 一律为 NFC（`unicode-normalization` crate）。扫描时把磁盘文件名转成 NFC；CLI 输入的路径、分支名、标签名在解析时也转成 NFC。
- **定位**（`RepoPath` → 磁盘路径）：逐个路径分量解析，每个分量依次尝试 NFC 形式、NFD 形式，都不存在时列出父目录、逐项比较 NFC 形式。只在本进程内缓存结果，不写入 `.lfv`：`.lfv` 需要跨设备复制，磁盘名映射换一台机器就没有意义了。
- **写入**：定位到已有条目时写到该条目；否则以 NFC 名字创建（缺失的父目录同样以 NFC 名字创建）。
- **规范化冲突**：同一目录下两个磁盘名转成 NFC 后相同 → 报错（spec §4.2）。扫描遍历目录时即可发现；定位时逐项比较也能发现。
- **大小写冲突**：写入路径 P 时，若同一目录已有一个与 P 仅大小写不同的条目，且对 P 做 `stat` 命中的正是该条目（说明该目录大小写不敏感），则报错并提示开启大小写敏感（spec §4.2）。扫描时发现两个活跃路径大小写折叠后相同，用同样的方法检测。按目录实测，而不是按平台推断，因为 Windows 可以对单个目录开启大小写敏感。
- **路径过长**：底层 I/O 因路径过长失败时，转换为说明原因的错误（spec §4.2）。

### 4.3 工作树扫描（惰性扫描）

若干命令（`lfv status`、`lfv track`、`lfv snap` 等）执行前会触发工作树扫描，对比磁盘与 `index.db`，更新 `tracked` / `untracked` 并驱动 §4.4–§4.6 的自动动作。实现为 `scan::Scanner::run(Mode::{Incremental, Full})`。

#### 4.3.1 扫描动作

先定义几个概念：

- 动态未跟踪：一个路径在 `config.yaml` 动态 untracked 列表中。
- 可跟踪候选：一个路径不在 `config.yaml` 动态 untracked 列表中。
- 状态更新：
  - 如果 LFV 可见且为动态未跟踪：设置为 `untracked` 状态；
  - 如果 LFV 可见且已有 tracked 状态行：对比 `disk_mtime_ns` / `disk_size` → 未变则沿用 `disk_hash`，变了则重算 hash → 与 `head_object` / `head_path` 比较得到 `modified` 或 `unmodified`（状态字母见 §2.7）；判断链条不一定要执行完才能出结果；
  - 如果 LFV 可见、未登记且为可跟踪候选：执行 auto-track（§4.4）；
  - LFV 不可见：删除状态表中对应记录，保持 `config.yaml` 与历史 Snapshot 不变，不产生 `D`。

| 层级 | 对象 | 扫描动作 |
| ---- | ---- | ---- |
| **A. 已登记路径** | `tracked` / `untracked` 中已有的行 | 对每行先依据 `.lfvignore` mtime 决定是否判断 LFV 可见，不可见则删除状态表记录，可见则 `stat` + 状态更新；`stat` 失败的 tracked 行走 auto-delete（§4.5）。成本 O(已登记路径数)。 |
| **B. 发现新路径** | 遍历中见到的、尚未登记的 LFV 可见路径 | 动态未跟踪文件设置为 `untracked`；可跟踪候选先做改名识别（§4.6），未配对者执行 auto-track（§4.4）。 |

以上扫描动作都会更新状态表。此外，`lfv track`、`lfv untrack` 也会更新 `config.yaml` 与状态表。

#### 4.3.2 增量扫描与失效

`scan_meta` 记录上次扫描完成时刻与规则文件 mtime。默认**增量**扫描，避免每次命令都全量递归整个工作树：

- **失效**（任一成立则层级 B 做受控全量遍历）：
  - `scan_meta` 为空（首次扫描或 `rebuild-index` 之后）；
  - `.lfvignore` 的 mtime 晚于 `scan_meta` 中记录值（遍历/剪枝边界变化）；
  - `config.yaml` 的 mtime 晚于记录值（按动态 untracked 列表与 LFV 可见路径**对账** `untracked` 行：存在则登记，不存在则删除）；
  - 用户执行 `lfv status --refresh`（或等价强制刷新）。
- **否则（增量）**：
  - 对**目录**：若目录 mtime ≤ `last_completed_at` 且该目录已在扫描登记中 → 不 descend；与 `.lfvignore` **目录剪枝**叠加（剪枝目录对 LFV 不可见）；
  - 对**文件**：若已在状态表且文件 mtime ≤ `last_completed_at` → 跳过层级 B 的「是否新路径」判定（层级 A 仍 `stat`）。
- **扫描结束**：更新 `last_completed_at`（取扫描开始时刻）及 `.lfvignore` / `config.yaml` 的 mtime。

> [!note] 注意
> 依赖 mtime 的增量策略在拷贝未保留时间戳、或文件系统秒级精度不足时可能漏扫；用 `--refresh` 兜底。

#### 4.3.3 触发时机

| 命令 / 场景 | 扫描 |
| ---- | ---- |
| `lfv status` | 默认增量扫描（§4.3.2） |
| `lfv status --refresh` | 强制失效后扫描 |
| `lfv track`（无参）、`lfv snap`（无参） | 扫描后再批量 track / snap |
| `lfv snap <file>`、`lfv mv`、`lfv delete`、`lfv rewind`、`lfv switch`、`lfv snap --tree`、`lfv rewind <TS-ish>` | 增量扫描（这些命令需要准确的 `modified` 判定） |
| `lfv status --include-untracked` | **不**改变扫描；仅多打印状态表中的 `untracked` 行（spec §4.3.2） |
| `lfv log`、`lfv show`、`lfv diff <FO-ish> <FO-ish>`、`lfv list`、标签/分支查询 | 不扫描 |

### 4.4 自动 track（新建文件）

扫描时发现工作树中存在一个 **LFV 可见**、且未进入状态表的路径，即：

- 若匹配 `.lfvignore` → **忽略**（不登记，§4.2）；
- 若在 `config.yaml` 动态 untracked 列表中 → 登记为 `untracked`（§4.2）；
- 若路径违反 spec §4.2 路径规则 → 不登记，只给出警告（扫描照常继续）并提示加入 `.lfvignore`；
- 否则 → 视为「OS 新建」，自动 `track`：`tracked::TrackedStore::create(path)` 分配 `file-id`、建目录并写 `meta.initial_path`，`tracked` 表插入 `status = modified`、无 `head_*` 的行。

**不会立即追加 Snapshot**，由后续 `lfv snap` 落盘首条 Snapshot。

自动 track 的触发时机：`lfv status`（扫描时即时执行）、`lfv track`（无参跟踪时执行）、`lfv snap`（无参批量拍照前执行）。

**取消 track**：`lfv untrack <file>` 写入 `config.yaml` 动态 untracked 列表；路径在盘上存在时同步登记 `untracked` 行（保留 file-id）。已有历史者可保证不再产生新 Snapshot；无历史的 auto-track 文件 untrack 后只删除跟踪缓存，以保证不追加 Snapshot。

路径在 config untracked 且盘上存在时，扫描时不会 auto-track；盘上消失则删除状态表行，文件再现时重新登记 `untracked`。

**同名文件的友好提示**：若某路径曾有一个已删除的 `file-id`（即该路径最新 Snapshot 的 `object = null`），自动 track 在同一路径上分配了新的 `file-id` 后，`lfv status` 会在该文件条目下附加提示：

```
  A   file:01HA7EEE...   docs/old-note.md  [newly tracked]
                      note: this path previously existed as file:023BHCA1 (deleted)
                      to continue its history instead, run:
                        lfv relink file:01HA7EEE --onto file:023BHCA1
```

查找方式：`snap_locator` 无法按路径反查，因此对新登记的路径在各 file-id 日志中查"最后拥有该路径且 HEAD object 为 null 的 file-id"；只在层级 B 发现新路径时做一次。该结果**不缓存**：`tracked` 表不设 `prior_file_id` 列，每次 `lfv status` 重新查找。这类 `A` 状态的文件很少，且首次 snap 后提示即消失，重查开销可以忽略；加一列缓存则要改 schema，并在 `rebuild-index` 中增加一条重算规则，不值得。

此提示仅在自动 track 产生新 `file-id` 时出现；一旦新 `file-id` 落下首条 Snapshot，它就是独立的文件，提示消失。

对于有历史的 untracked 文件，由于它的最新 Snapshot 仍指向某个 Object，当再次人工 `lfv track <file>` 时，LFV 会先从 `config.yaml` 中的动态 untracked 列表中移除，再按 `untracked.file_id` 记录的原 `file-id` 进入跟踪（无记录时从日志反查），并在状态表中更新为 tracked 行以缓存。注意，这里与 delete 文件后 `lfv track <file>` 的算法不一样，后者一定默认产生新的 `file-id`，除非人为地 `lfv revive`。

### 4.5 自动 delete（OS 删除文件）

扫描时发现某个**跟踪文件**的路径在工作树上消失，且没有证据表明是改名（即改名自动识别未能配对），视为"OS 删除文件"。LFV **不立即**追加 Snapshot，而是把 `tracked` 行置为 `present = 0, status = modified`；因为当前路径在 FS 上不存在，`lfv status` 会显示为 `D`。

后继执行 `lfv snap` 命令时，会落地状态的更新（§4.9）。

这样设计的好处：`lfv snap` 之前，`D` 文件仍在跟踪列表中，用户还有机会通过 `lfv mv file:<old> <new-path>` 将其识别为改名，避免误删。

### 4.6 rename / move（lfv mv）

**`lfv mv` 只做路径操作**：`<src>` 接受路径或 file-id（按 file-id 定位时取其当前路径，适用于 `D` 行这类盘上已消失的文件）；`<dst>` 只允许路径，不接受 file-id。用于文件在磁盘上仍存在（或刚被 OS 移动）时的改名/移动。不允许 `<dst>` 为 file-id 的主要原因是：它会造成心智混乱——用户可能误以为保留的 file-id 是 `<dst>` 那一侧。若需要把历史续接到另一个 file-id，请使用 `lfv relink`（见 §4.7）。

- `lfv mv <src> <dst>`（`ops::mv`）：把 src 对应的跟踪文件迁移到 dst 路径。
  - 追加快照前，先按 src 当前内容（即移动后 dst 的内容）算出 hash 并做环回检查（§4.13）：命中则拒绝整条命令，此时盘上文件尚未移动。
  - 若工作树上 src 路径仍存在且 dst 不存在，CLI 会先把文件移到 dst，再追加快照（原子语义）。
  - 若 src 路径已不存在（OS/编辑器先动了），dst 路径上已有该文件，则 `lfv mv` 只做"登记快照"，不再动盘。
  - 新 Snapshot 的 `object` 由 dst 当前内容的 hash 决定：若与父快照相同则呈现为纯 `R`，不同则呈现为 `R+M`。
  - `message` 默认为 `rename: <old-path> -> <new-path>`。
- **自动识别 vs. 手动登记**（`scan::rename_detect`）：
  - 自动识别仅在内容哈希 **完全一致** 时生效（`rename.autodetect`，默认开启）。只有存在 `present = 0` 的 tracked 行时才对新发现路径计算 hash；`hash == head_object` 且配对**唯一**（该 hash 恰好一个消失行、一个新路径）→ 把该 tracked 行的 `path` 改为新路径、`present = 1`，不新建 file-id；`lfv status` 直接以 `R` 行显示。
  - 零字节文件不参与识别；多对多（多个消失行与多个新路径 hash 相同）不配对，按 `D` 与 `A` 分别列出，由用户用 `lfv mv` 自行声明移动关系。
  - 若 OS 移动文件 **并且** 内容也变了，自动识别会失败——`lfv status` 会同时显示一条 `D` 行（旧 file-id 在原路径消失）和一条 `A` 行（新路径上的新 file-id）。用户检视后用 `lfv relink <new-file> --onto <old-file>` 显式声明历史续接。
  - 自动识别也可被关闭（适用于成批改名 + 编辑的场景，避免误配对）。
- 历史回溯：`lfv log <FS-ish>` 会以"`R` 旧路径 -> 新路径"形式渲染 rename 事件，与 `A`(add) / `M`(modify) / `D`(delete) / `R+M`(rename+modify) 并列（见 §3.3.1）。

### 4.7 历史续接（lfv relink）

`lfv relink <src-file-id> --onto <dst-file-id>` 用于将 src-file-id **当前分支**的历史在 dst-file-id **当前分支**的末尾续接，同时盘上 src-file-id 对应文件将以后与 dst-file-id 绑定。**仅操作两个 file-id 各自当前分支，不涉及其他分支，不做 4-way merge 内容合并。**

该命令专为"用户误操作导致 LFV 产生了新的 file-id"的场景设计（典型场景：OS 层改名+改内容导致 auto-track 分配了新 file-id），不应作为日常命令。如果 src-file-id 还没有历史，则 dst-file-id 的历史维护不变，只做 file-id 绑定转移。

- **src-file-id**：状态 `A`/`M`，盘上存在的活文件（新 file-id，尚在跟踪中）
- **dst-file-id**：状态 `D`，已从磁盘消失的文件（旧 file-id，待续接），或最新快照 `object = null` 的已删除文件
- **最终保留的 file-id 是 `--onto` 的那一侧（dst-file-id）**，与介词方向一致

执行后 src-file-id 的当前路径和内容作为 dst-file-id 历史的下一条 Snapshot 追加；src-file-id 的 `tracked` 行改挂到 dst-file-id（`file_id` 换、`head_*` 取 dst），不再产生新快照。**src-file-id 的 `files/<src-file-id>/` 目录及 `snapshots.log` 完整保留**（append-only，不删除），并在其 `meta.yaml` 写入 `retired: {at, onto: dst}`；src-file-id 成为"已退役"的 file-id，`lfv log src-file-id` 仍然有效，`lfv list` 默认不列。

**情况一：src-file-id 尚无快照历史**

src-file-id 只有跟踪记录，未产生任何 Snapshot。执行后：将 src-file-id 的当前路径的文件使用 dst-file-id 这个 file-id。事件类型按 §3.3.1 由 dst-file-id 末尾快照与当前的 path/object 对比推导（通常为 `R+M` 或 `R`，dst 已删除时为 `+`）。

**情况二：src-file-id 已有快照历史**

src-file-id 已经产生了若干 Snapshot（链为 `snap_A1 -> snap_A2 -> ... -> snap_An`）。执行后，将 src-file-id 当前分支的快照链复制并追加到 dst-file-id 当前分支历史末尾：

- 每条快照新生 ULID（`snap_B1 ... snap_Bn`），`digest` 对新字段重算
- `snap_B1.parent` = dst-file-id 的最新快照；`snap_Bx.parent` = `snap_B(x-1)`
- `snap_B1` 的事件类型由 dst-file-id 末尾快照与 `snap_B1` 的 path/object 对比推导；`snap_B2` 之后与原 src-file-id 内部推导结果一致，不受影响
- 接缝处的事件类型可能呈现为跨路径的跳变，这是用户主动声明历史续接的预期代价
- 复制前对每条待追加快照做环回检查（§4.13）：任一条命中则整个 relink 拒绝，不写入
- 同样，当前盘上 src-file-id 对应文件将以后与 dst-file-id 绑定

### 4.8 lfv delete

`lfv delete <file>`（`ops::delete`）：盘上存在则删除该文件；`tracked` 行置 `present = 0, status = modified`。后继 `lfv snap` 时，LFV 会参照目标文件的当前 FS 存在状态：若文件不存在，则追加 `object = null` 的 Snapshot，并删除该 `tracked` 行。

此后该 file-id 不再有 `tracked` 行。它的最后路径保留在该 Snapshot 中，该路径仍可解析到它（回退到历史中最后拥有该路径且未退役的 file-id，见 §2.8），但一旦被新文件占用就解析到新 file-id；`lfv list --deleted` 仍可按路径显示。

文件 **历史完整保留**：仍可 `lfv log` / `lfv show` / `lfv diff` 查询；想"复活"用 `lfv revive`（§4.10）或直接 `lfv rewind <file> <snap>`，二者都会自动新建保留分支（不破坏既有删除事件）。

### 4.9 `lfv snap`：内部执行流程

**单文件**（`ops::snap_file`）：

1. 读 `tracked` 行；`status = unmodified` → 跳过。
2. `present = 1`：
   - `object.put_path(path)` → `(hash, size)`；
   - `hash == head_object && path == head_path` → 拒绝（未变化）；
   - `head_snap` 为 `None`（`A`）→ 首条快照，`parent = null`；
   - 否则 `snapshot::loop_check`（§4.13）→ 命中则拒绝并输出 rewind 提示；
   - append `Snapshot { parent: head_snap, path, object: hash, size }`；分支表推进；`tracked` 行更新 `head_*`、`disk_*`，`status = unmodified`。
3. `present = 0`：
   - `head_snap` 为 `None`（`A` 且已消失）→ 不产生快照，删除 `tracked` 行，`files/<id>/` 目录保留为空壳（gc 可清理）；
   - 否则 append `Snapshot { parent: head_snap, path: 盘上最后已知路径, object: null, size: 0 }`；分支表推进；删除 `tracked` 行。
4. 写入顺序按 §3.9。

**批量**（`ops::snap_all`）= 扫描 + 对每个 `status = modified` 行调用 `snap_file`，逐文件独立事务，单个失败不回滚其它，最后汇总报告。省略 `<file>` 的 `lfv snap -m <msg>` 与显式的 `lfv snap --snap-all -m <msg>` 是同一入口：都对当前所有 `modified` 文件应用同一条 `<msg>`；`--snap-all` 只是让"批量、共用一条 message"这个意图在命令行上可见，不是独立算法。`--snap-all` 与 `--tree` 互斥，由 CLI 层拒绝同时传入（§4.11）。

### 4.10 `lfv rewind <file>`、`lfv switch`、`lfv revive`

**`ops::rewind_file(file, target: SnapId, opts { ff_first })`**（spec §4.5）：

```
head = branches[HEAD]
loop_mode := worktree hash == target.object && target is a strict ancestor of head && head.object != target.object
if worktree modified && !loop_mode              -> error (run `lfv snap` first, or discard changes)
if ff_first && some branch B has head object == target.object   // tree rewind only
    switch to B (prefer current, else newest); return FF
preserve: new branch <kind>/<anchor-short>/<n> -> head          // kind: rewind | detour | rebase | revive
if loop_mode:                                                   // §4.13
    anchor = first snapshot of the same-object run that contains `target`
    append Snapshot { parent: anchor.parent, object: target.object, path: current path }
    branches[HEAD] = new snapshot
else:
    branches[HEAD] = target
    object.restore_to(target.object, current path)              // content only; path unchanged
index: tracked.head_* / disk_* refresh; status re-derived per §2.7
       (content now equals target.object => modified/R when target.path != current path, else unmodified)
```

普通模式与 loop 模式共用"保留旧 HEAD 到新分支"这一步，只是保留分支的 `kind` 不同（`rewind` / `detour`）。

**`ops::switch(file, branch)`**：文件 `modified` → 拒绝；目标分支 HEAD `object != null` → `restore_to` 到当前路径，`tracked.head_*` 刷新；HEAD `object == null` → 从工作树删除文件、删除 `tracked` 行；写 `HEAD` 文件。

**`ops::revive(file, target?)`**：`target` 缺省为当前分支最后一条 `object != null` 的快照；调用 `rewind_file`，`kind = revive`；内容写回 `target.path`（最后已知路径），并插入 `tracked` 行（`present = 1, unmodified`）。若该路径已被另一个活跃 file-id 占用则拒绝。

### 4.11 `lfv snap --tree`：内部执行流程

`--snap-all` 与 `--tree` 是互斥的两个 flag，CLI 层直接拒绝同时传入两者（spec §4.3.3）；`--snap-all` 走的是 `ops::snap_all`（§4.9 的批量 snap，只是显式命名并要求所有 `modified` 文件共用同一条 `-m` message），不进入 `ops::tree_snap`。`lfv snap --tree`（`ops::tree_snap`）通过 spec §4.3.3 的前置条件（即：调用时刻已无任何 `modified` 状态的跟踪文件，通常是先跑过一次 `lfv snap` 或 `lfv snap --snap-all`）后：

1. 从 `tracked` 表读取所有 `present = 1, status = unmodified` 行的 `head_path` / `head_object`（不读各文件日志），构建 Tree Object 清单（按路径排序）。
2. 规范化序列化后计算 blake3 hash，得到 tree-id；若 `objects/` 中已有该对象则复用，否则写入。
3. 向 `.lfv/trees/snapshots.log` append 一条 Tree Snapshot 记录，parent 指向当前 tree HEAD（若为空则为 null）。
4. 更新 `.lfv/trees/HEAD` 为新 Tree Snapshot 的 id。
5. 若指定 `--tag`，向 `.lfv/trees/tags.yaml` 写入标签。
6. 展开清单，向 `index.db` 的 `tree_file_refs` 表逐 file 插入一行（file-id 来自步骤 1 的 `tracked` 行），并写 `snap_locator`。

### 4.12 `lfv rewind <TS-ish>`：内部执行流程

`lfv rewind <TS-ish>`（`ops::tree_rewind`）通过 spec §4.5.1 的前置校验后：

1. 若当前 tree HEAD 无标签且不是目标的祖先，写入 `tree:detour/<anchor-short>/<n>` 标签。
2. 将 `.lfv/trees/HEAD` 更新为目标 Tree Snapshot 的 snap id。
3. 读取目标 tree-snapshot 的 Tree Object，得到 `{path, file-object-hash}` 清单；经 `tree_file_refs(tree_snap_id = target)` 把每个条目映射到 file-id。
4. 按以下三类分别处理所有 file：

**分类 A：在清单中、file-id 仍活跃（有 `tracked` 行）的 file**，对每个 file 以 `target_hash` 为目标，执行 FF 优先 rewind（`rewind_file(ff_first = true)`）：
  - **FF 路径**：若某分支的 HEAD object hash == `target_hash` → 优先选当前分支；无则选最近创建的匹配分支 → 直接 `switch`，不新建分支。提示：`[FF] docs/note.md → branch main`
  - **rewind 路径**：否则 → 执行标准 rewind，自动新建保留分支。提示：`[rewind] docs/note.md → kept old HEAD on rewind/7RQ2M9KA/1`
  - 两种路径之后，若文件当前路径 ≠ 清单路径，把盘上文件移到清单路径并更新 `tracked.path`（下次 snap 记 `R`）。

**分类 B：有 `tracked` 行、但不在清单中的 file**（tree-snapshot 后新增并已快照过的 file）：
  → 仅删除工作区文件，不动 `config.yaml`。后续扫描（§4.5）自动标记为 `D`。提示：`[deleted] docs/new-chapter.md`

**分类 C：在清单中、但 file-id 当前无 `tracked` 行（已 untracked、已删除、已退役）或 `tree_file_refs` 无法映射的条目**：
  → 仅将文件字节写回清单路径，不修改任何状态表或 config。后续扫描接管：
  - 原 file-id 为 deleted：走 §4.4 同名文件友好提示，用户可 `lfv relink` 续接历史。
  - 原 file-id 为 untracked：扫描后显示 `~`，用户自行决定是否 `lfv track`。
  提示：`[restored] docs/old-chapter.md`

5. 输出汇总。

### 4.13 环回检测与处理

为保证每个文件的历史在单条分支上呈现**无环的内容演化 DAG**，LFV 强制维护**分支对象唯一性**不变量。目前由 `lfv snap` 与 `lfv verify` 触发检测；`lfv mv`、`lfv merge`、`lfv rebase`、`lfv relink` 追加快照时同样遵守（处理方式见 spec §4.7.4、本文 §4.7）。

- **在任意一条分支上，同一个 `file-object-hash` 只能构成一段连续的快照 run，不允许非连续地再次出现。** 连续同 object 的快照（纯改名、路径改回）在内容层折叠为同一节点；`null` 不参与唯一性，但会打断 run：`O1 → null → O1` 属于 O1 非连续再次出现。
- **历史完整性优先**：绝不静默丢弃或重写中间历史，用户必须显式执行 `rewind` 才能"跳过"重复段。
- **分支清洁**：环回会污染当前分支，必须将中间历史移到 `detour/<anchor-short>/<n>` 保留分支，用户可审阅后决定是否删除。
- **与 `rewind` 命令语义一致**：`lfv rewind` 本身就是用于"回到过去并自动建分支"，唯一性不变量正是为此场景设计。

#### 4.13.1 检测时机

**`lfv snap` 触发**：执行 `lfv snap <file>`（或批量 `lfv snap` 中对单个文件处理）时，在计算新快照的 `object` 字段后、写入 `snapshots.log` 之前，调用 `snapshot::loop_check(log, head, new_object)`：**沿着当前分支的 parent 链向上回溯**，先跳过 object 与新快照相同的连续前缀（同一 run），之后若再遇到 object 相同的祖先 → 返回该祖先所在 run 的首条快照 `ancestor_snap`。

- 返回 `None` → 正常追加新快照。
- 返回 `Some(ancestor_snap)` → 触发**环回事件**，拒绝本次 `lfv snap`，并输出提示。

#### 4.13.2 环回事件的处理

LFV 不自动重写历史，而是提示用户使用 `lfv rewind` 来完成回溯，同时保留中间的所有历史：

```
warning: content of docs/note.md matches ancestor snap:01HXYZ on branch main
         (object hash blake3:abc123...)
suggestion: run `lfv rewind docs/note.md snap:01HXYZ`
            this will create a new branch anchored before the duplicate,
            preserving all intermediate history as a detour branch.
```

用户执行 `lfv rewind docs/note.md snap:01HXYZ` 后，`rewind_file` 识别出 loop 模式（工作区内容 == 目标 object、目标是 HEAD 的严格祖先、HEAD object ≠ 目标 object），于是：

1. 新建分支 `detour/<anchor-short>/<n>` 指向当前 HEAD（`anchor-short` 取旧 HEAD 的 ULID 末 8 位，重名则 `n` 递增）；分支名 `main` 本身不动；
2. 在 `main` 分支上创建一条**新快照**，其：
   - `parent` 指向 `ancestor_snap` 的**父快照**（`ancestor_snap` 为匹配 run 的首条，因此其父 object 必不同）；
   - `object` 等于新文件内容（即触发环回的那个 `object`）；
   - `path`、`message` 等按正常快照记录。
3. 将 `main` 的 HEAD 指向这个新快照，工作区内容保持不变（已是新内容）。

这样，原分支上的中间历史（`ancestor_snap` 之后到当前的所有快照）全部被保留在 `detour/...` 分支中，而 `main` 分支则"跳过"了那段历史，直接接续到更早的状态，且**不会在 `main` 上产生非连续的重复 object hash**。

#### 4.13.3 对 `lfv verify` 的校验要求

`lfv verify` 命令必须遍历每个文件的所有分支，检查是否存在同一分支上非连续地出现相同 `object` 的记录。若发现，报告为**数据完整性错误**（非警告），因为正常操作流程下不应发生（环回事件已被拒绝并引导用户使用 `rewind`）。

```bash
error: branch 'main' of file 'docs/note.md' contains duplicate object hash:
       snap:01HXYZ (object blake3:abc123...)
       snap:02HABC (object blake3:abc123...)
       This violates the single-branch object uniqueness invariant.
```

### 4.14 `merge::replay`（merge / rebase / --pick）

```rust
pub struct ReplayState {          // persisted as REPLAY.yaml while in progress (§3.7)
    kind: Merge | Rebase | Pick,
    file: FileId, branch: BranchName,
    original_head: SnapId, preserved_branch: Option<BranchName>,
    steps: Vec<Step { base: Option<ObjectHash>, theirs: ObjectHash, theirs_snap: SnapId, path: RepoPath }>,
    next: usize,
}
```

- 定位同的祖先（spec §3.3）：对 ours 链的每个祖先 object 建 `HashSet`，沿 theirs 链找首个命中；命中的 object 是 `base-object`，两侧各自最近的持有者是 `base-snap-ours/theirs`。
- 每步：`merge::three_way(base, ours, theirs)`（`diffy::merge`，冲突标记行首替换为 spec §4.7.3 的标签）；内容层是三路合并，"4-way"指两个 base 快照可以不同。文本判定与 diff 共用：无 NUL 字节且 UTF-8 可解码。
- 无冲突 → `put_bytes` → `loop_check` → 环回则暂停（spec §4.7.4）→ append → 推进分支 → `next += 1`。
- 冲突 → 文本：写冲突标记到工作区；二进制：不改工作区，提示 `--continue --ours|--theirs` → 保存 `REPLAY.yaml` → 退出码非 0。`--continue`：读工作区（或所选一侧）为新 object 继续；`--abort`：分支指回 `original_head`，`restore_to`，删 `REPLAY.yaml`（中间快照悬空，由 gc 回收；`preserved_branch` 保留）。
- rebase = 先 `rewind_file(base-snap-ours, kind = rebase)` 再把 theirs 步骤 + ours 原步骤依次入队；merge = 只入队 theirs 步骤；`--pick` = 单步入队，`base = pick.parent.object`（父为 null 或父 object 为 null 时 base 为空内容），且完全跳过"定位同的祖先"这一步（§3.3 的祖先查找只有 merge/rebase 需要）。三者除 base 计算方式与步数不同外，复用同一套 replay：冲突标记、`REPLAY.yaml` 持久化、环回检测、`--continue`/`--abort` 完全一致；`--pick` 暂停后同样用 `lfv merge --continue` / `lfv merge --abort` 续接或回滚，`kind = Pick` 只用于状态展示，不引入新命令。
- `--continue` / `--abort` 省略 `<file>` 时（spec §4.7.3）：遍历 `files/*/REPLAY.yaml`，恰好一个即作用于该文件；多个则报错要求指定 `<file>`；零个报错。

### 4.15 `lfv verify`

`ops::verify` 顺序执行，全部跑完再汇总（不在第一个错误处停止）：

1. 对象：`object.iter` + `open` 重算 hash；同 hash 双扩展名并存；
2. 快照：每行 digest 重算（§3.3.2）；`parent` 存在于同一日志；
3. 分支/标签表指向的快照存在；`HEAD` 分支存在于分支表（或日志为空）；
4. 每条分支的对象唯一性（§4.13.3）；
5. Tree Object 清单可解析、条目指向的对象存在；Tree Snapshot 的 object 存在；
6. `index.db` 与真理源抽样一致性（分支表、`snap_locator`），不一致只提示 `lfv rebuild-index`。

### 4.16 `lfv import`：内部执行流程

`lfv import <archive>`（`ops::import`）与 `lfv export`（`ops::export`，spec §4.8）互为逆操作，且不触碰工作树、状态表：

1. 解压/打开归档，定位其 `<ULID>/` 目录；若当前仓库 `files/<ULID>/` 已存在则报错拒绝（正常不会发生，ULID 全局唯一），不做部分导入或合并两条历史。
2. 解析归档内 `snapshots.log`，逐行重算 `digest`（§3.3.2）并与记录比对；任一行不符即整体拒绝导入。
3. 遍历该日志引用到的每个 `object` hash：归档同时带着这些 File Object（结构与 `.lfv/objects/` 一致），对每个 hash 若本地 `objects/` 已存在则跳过（按内容去重，与 §3.2 的写入策略一致），否则原样写入（不重新压缩判定，信任归档里的编码；`verify` 会在之后校验 hash）。
4. 把归档的 `meta.yaml`、`HEAD`、`branches.yaml`、`tags.yaml`、`snapshots.log` 整体拷贝到 `.lfv/files/<ULID>/`。
5. 为该日志的每条快照写入 `snap_locator(snap_id, file_id)`，使 `lfv log <snap-id>` 立即可用；**不**创建 `tracked` 行，也不修改 `config.yaml`——该 file-id 处于"有历史、未激活"状态，等同于 §4.7 relink 中"已退役"的 file-id，但没有 `meta.retired`（它不是被某个 dst 吸收，只是尚未 materialize）。
6. 输出汇总：file-id、分支数、快照数、导入的对象数。提示可用 `lfv revive <file-id>` 恢复到工作树。

`lfv revive` 对无 `tracked` 行、`HEAD` 快照 `object != null` 的 file-id 同样适用（§4.10 revive 的前提只要求"能定位到一个非 null 快照"，不要求文件此前处于跟踪状态）；恢复路径已被另一个活跃 file-id 占用时按 §4.10 拒绝，用户可先 `lfv mv` 让路或改用其它路径。

### 4.17 流程 → 模块映射总表

| 流程 | 触发命令 | 入口 | 主要依赖 |
| --- | --- | --- | --- |
| 可见性规则 §4.2 | 所有扫描 | `scan::visibility` | `repo`（`.lfvignore`、`config.untracked`）、`ignore` crate |
| 惰性扫描 §4.3 | §4.3.3 表 | `scan::Scanner::run` | `index`、`tracked`、`object`（hash） |
| auto-track §4.4 | 扫描 | `scan::autotrack` | `tracked::TrackedStore::create`、`index` |
| auto-delete §4.5 | 扫描 | `scan` 层级 A | `index` |
| 改名识别 §4.6 | 扫描 | `scan::rename_detect` | `index`、`object`（hash） |
| mv §4.6 | mv | `ops::mv` | `resolve`、`object`、`snapshot`、`tracked`、`index` |
| relink §4.7 | relink | `ops::relink` | `snapshot`（复制链）、`tracked`（retired）、`index` |
| delete §4.8 | delete | `ops::delete` | `index`、`util::fs` |
| snap §4.9 | snap | `ops::snap_file` / `ops::snap_all` | `object`、`snapshot::loop_check`、`tracked`、`index` |
| rewind / switch / revive §4.10 | rewind, switch, revive | `ops::rewind_file` / `ops::switch` / `ops::revive` | `snapshot`、`object::restore_to`、`tracked`、`index` |
| snap --tree §4.11 | snap --tree | `ops::tree_snap` | `object::TreeManifest`、`tree`、`index` |
| rewind <TS-ish> §4.12 | rewind tree: | `ops::tree_rewind` | `tree`、`index.tree_file_refs`、`ops::rewind_file` |
| 环回 §4.13 | snap / mv / verify / merge / rebase / relink | `snapshot::loop_check` | — |
| merge / rebase / --pick §4.14 | merge, rebase, --continue, --abort | `merge::replay` | `diffy`、`ops::rewind_file`、`snapshot`、`object` |
| verify §4.15 | verify | `ops::verify` | 全部存储层 |
| import §4.16 | import | `ops::import` | `snapshot`（digest 校验）、`object`（去重写入）、`tracked`（meta/branches/tags 拷贝）、`index.snap_locator` |
| gc §3.11 | gc | `ops::gc` | `snapshot::prune`、`object::remove` |
| rebuild-index §3.10 | rebuild-index | `ops::rebuild_index` | `index`、`tracked`、`tree` |
| log / show / diff / list / branches / tags | 查询类 | `ops::query::*` | `resolve`、`snapshot`、`diff`、`index` |

## 5. 路线图落地顺序（对照 spec §7）

| 版本 | 命令 | 需要的模块 |
| --- | --- | --- |
| v0.1 | init / track / snap / log / status / show | `util repo object snapshot reftable tracked index scan resolve ops(snap, track, log, status, show)`；**`index` 的全部 schema 与 `rebuild-index` 一起在 v0.1 落地**，否则后续无法验证"可重建"这一核心承诺 |
| v0.2 | diff / branches / rewind / switch | `diff`、`ops(rewind_file, switch)`；`mv` / `delete` / `revive` / `relink` 一并归入 v0.2（只依赖 v0.1 模块） |
| v0.3 | tag / export / import / gc / verify | `ops(gc, verify, export, import)` |
| v0.4 | merge / rebase / --pick | `merge` |
| v0.5 | tree plane | `tree`、`ops(tree_snap, tree_rewind)`、`tree_file_refs` |
