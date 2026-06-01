# LFV 轻量文件中心版本控制

> 本文件是 LFV 项目的中文版设计文档。英文版请见 `design.md`。
> 入口文档 `doc/design.md` 用于英文版设计文档及其索引，包括进入中文版。

---

## 1. 项目背景与动机

`git` 是非常优秀的版本控制工具，但它的设计目标是 **以仓库为单位** 的版本管理，最适合一组逻辑相关的源代码集合：一次提交可以横跨多个文件，分支记录的是整个工作树的整体演化。

然而日常工作中还存在另一类典型场景，它具有以下特征：

- 每个文件 **逻辑独立**，相互之间没有耦合；
- 每个文件 **自有一条演化历史**，与其它文件无关；
- 没有"一次性提交多文件"的语义需求；
- 用户关心的是 **某一个具体文件** 在时间轴上的演化、对比、回退。

典型例子：

1. **NAS 文件同步系统**：每个备份文件都有自己的历史，但不需要跨文件的"原子提交"。
2. **一对一笔记仓库**：每一篇笔记独立演化，A 笔记的修改不应"污染"B 笔记的历史。
3. **设计稿/合同/单文档档案库**：每个文档单独跟踪版本。
4. **配置文件目录**：每个配置文件独立演进，互不影响。

目前业界对这类场景的处理普遍不令人满意：

- **简单备份**：仅复制旧版本，无法直观看到每个版本的目的；
- **保留 N 个历史版本**：缺乏语义信息，无法 diff、无法标注；
- **整目录 git 仓库**：把"无关文件"的演化混在一起，历史线被淹没，违背"以文件为中心"的心智模型；
- **云盘历史版本**：通常不可离线使用，也不支持分支、对比、可移植。

## 2. 项目目标

LFV (Lightweight File Versioning) 的目标：

1. 在指定的工作目录下建立 `.lfv` 目录，作为该目录（含所有子目录）下被跟踪文件的 **本地版本仓库**。
2. 提供 `lfv` CLI，使用户能像使用 `git` 那样直观地：
   - 选择性地跟踪某个文件；
   - 为该文件创建带注释的快照；
   - 查看该文件的历史快照列表；
   - 对比任意两个快照（或快照与当前内容）；
   - 回溯（rewind）到某个历史快照——回溯时 **自动建立新分支**，绝不破坏既有历史。
3. 每个文件拥有 **独立的**：历史线、分支集合、标签集合。
4. 提供良好的可移植性：`.lfv` 目录可以随工作目录一起拷贝/同步，迁移后行为一致。
5. 数据存储紧凑：通过内容哈希去重和压缩，避免简单备份导致的存储膨胀。

## 3. 非目标（明确不做什么）

为了保持"轻量"，下列内容 **不在** 本项目范围内：

- **多文件原子提交**：每次快照只针对单个文件，不存在"一次提交多个文件"的语义。
- **分布式协作**：不实现 push / pull / remote / merge 等多用户协同语义。`.lfv` 通过外部同步（如 NAS、网盘）共享即可，但不解决并发冲突。
- **完整的合并算法**：不为同一文件的两个分支提供自动 3-way merge；用户决定保留哪个分支即可。
- **替代 git**：在源代码工程场景下应继续使用 git；LFV 只服务"文件中心"场景。
- **图形界面**：1.0 版本只提供 CLI；GUI 列为后续可能的扩展。
- **暂存操作**：一律"工作区即快照应用部分"。

## 4. 核心概念

### 4.1 基本术语

| 概念 | 英文 | 说明 |
| ---- | ---- | ---- |
| 仓库 | Repository | 工作目录下的 `.lfv` 目录，承载所有被跟踪文件的元数据与对象存储。 |
| 工作树 | Working Tree | 工作目录本身（除 `.lfv` 外），用户实际操作的文件位于此处。 |
| 跟踪文件 | Tracked File | 由 `lfv track` 加入仓库管理的文件。内部以 `file-id`（ULID）作为稳定身份，与路径完全解耦（见 §4.5）。 |
| 文件状态 | File Status | 对已纳入 LFV 视野的路径，状态为 `unmodified`、`modified` 或 `untracked`。（见 §6） |
| 对象 | Object | 内容寻址的文件内容或全局引用存储单元；按内容哈希去重，不同文件可能共享同一对象。 |
| 快照 | Snapshot | 某个文件或全局文件在某一时刻的拍照：内容指针 + 路径 + 元数据（消息、时间戳、作者、父快照）。|
| 分支 | Branch | 某个跟踪文件下的一条快照链；默认分支 `main`。每个文件的分支命名空间彼此独立。 |
| 头部 | HEAD | 某个跟踪文件当前所在的分支名与最新快照指针。 |
| 标签 | Tag | 对某个快照的可读命名（可选），用于稳定地引用某个版本。 |
| 动作 | Action | 改变文件状态的操作。显式动作：`track`、`snap`、`untrack`；隐式动作：`modify`（用户编辑文件）、`auto-track` / `auto-delete`（LFV 扫描时自动响应 OS 事件，见 §6）。 |
| LFV 可见 | — | 在 `.lfvignore` 匹配目录剪枝后，工作目录中剩余的文件。 |

### 4.2 存储对象：File Object、Tree Object、File Snapshot、Tree Snapshot

LFV 的存储对象有两种 **完全不可变、内容寻址** 的类型 —— File Object、Tree Object —— 组成，均存放于 `.lfv/objects/`，按内容 hash 去重。它们共同点是，写一次，永不修改。可通过 `lfv verify` 重算 hash 校验完整性。

LFV 的快照是 **append-only 的事件记录层**，以 ULID 寻址，记录"某个时刻发生了什么"。它也分两种类型：File Snapshot、Tree Snapshot，结构高度对称，它们分别用于**两个独立视图**。

所有快照 append-only，不会重写已有快照。

#### File Object

File Object 存储单个文件的原始字节（压缩后落盘），它的 file-id 为文件原始内容的 `blake3`。不同路径或历史的文件，若内容相同，共享同一个 File Object，节省空间。

其它存储方面的信息见 §5.1。

`file-id` 是某个跟踪文件在仓库内部的**稳定身份标识**：

- **首次跟踪时**分配一个独立的 ULID，例如 `f_01HXYZABC...`。
- 与文件路径**无任何派生关系**，不会因改名、移动而变化；文件删除后 `file-id` 依然有效。
- 用户在 CLI 层面通过**当前路径**定位文件，CLI 内部经由 `file_states` 表中的 `fullpath → file_id` 索引解析；任何接受 `<file>` 的命令同时接受路径或 `f_*`。
- 文件的"当前路径"不存放于 `meta.yaml`，而由状态表维护并随 Snapshot 链的事件派生更新——仓库始终只有一份真理源（Snapshot 链），可变状态可重建。

#### Tree Object

对用户而言，这是全局快照。

它存储工作目录某一时刻所有有效跟踪文件的内容清单：按路径字典序排列的 `{path, file-object-hash}` 条目列表，按指定顺序形成规范化清单。tree-id 为该清单字节串的 `blake3()`，内容寻址。两次工作目录状态完全相同，产生同一 Tree Object hash，`objects/` 里只存一份。

- 清单只包含当前有有效（非 null）object 的跟踪文件；工作目录为空时清单为零字节，仍是合法的 Tree Object。
- 通常体积很小（每个跟踪文件一行），以 `.raw` 原样存储。
- 与 File Object 共用同一 `objects/` 目录，去重机制完全一致。

#### File Snapshot

每条 File Snapshot 是某个跟踪文件在某一时刻的完整事件记录，核心字段：

- `id`：`snap_<ULID>`，时间有序、全局唯一，不可重写。
- `parent`：父快照 id；分支首条为 `null`。
- `path`：此次快照时文件在工作树上的相对路径（路径是快照的字段，不是文件的身份）。
- `object`：所引用 File Object 的 blake3 hash；文件不存在时为 `null`（delete 事件）。
- `digest`：对本条记录除 `digest` 外所有字段做规范化序列化后的 blake3 hash，供 `lfv verify` 防篡改校验。

此外其它信息字段，消息、作者、时间等，参看 §5.2。

File Snapshot 有以下特点：

- **不显式存事件类型**
- **不记录所属分支**，快照只记录历史事件。
- 整个文件生命周期就是 File Snapshot 链。
- "重命名/移动"、"新增"、"内容变更"、"删除"仅在用户界面上有意义，靠对父快照字段比对即可区分。由读侧根据 `(parent.path, parent.object, path, object)` 四元组推导（add / modify / rename / rename+modify / delete / revive），详见 §5.2.1。
- 每条 File Snapshot 只有一个 parent 指针，整体构成**有向树（森林）**：分支出去即独立演化，不存在拓扑意义上的合并点。

用`lfv log docs/note.md`查看文件历史视图，沿该 file-id 的 File Snapshot 链游走；`lfv log` 在文件历史节点旁附加 tree 关联信息（来自 `index.db` 的 `tree_file_refs` 表）。

#### Tree Snapshot

每条 Tree Snapshot 是整个工作目录的里程碑记录，主要存储以下内容：

- `id`：`snap_<ULID>` 仅 append，不重写
- `object`：tree-object 指针，指向一个 Tree Object（永不为 null）
- `parent`：父快照 id
- `digest`：对本条记录除 `digest` 外所有字段做规范化序列化后的 blake3 hash，供 `lfv verify` 防篡改校验。

其它字段如消息、作者、时间等请参看 §5.3。

Tree Snapshot 与 File Snapshot 结构对称，但字段有以下差异：

- `path` 字段为空（tree 不需要定位路径）。
- `object` 不可为 null（tree 快照始终指向一个 Tree Object）。

tree 层没有分支概念，因此也没有 `branches.yaml`。

全局快照对用户来说可选：仓库可以没有任何 Tree Snapshot。

Tree Snapshot 同样只有一个 parent 指针，构成有向树，存放于 `.lfv/trees/snapshots.log`（固定路径）。

用 `lfv log --tree` 查看全局历史视图，沿 Tree Snapshot 链游走（见 §7.3）。

从一个 Tree Snapshot 出发，可通过 Tree Object 清单里的 file-object-hash 定位到某个文件在那个时刻的内容；进而在该文件的 Snapshot 链中查找 object hash 匹配的条目，即可找到对应的 File Snapshot（单分支唯一性不变量保证每条分支上至多有一个候选）。

#### 两层拓扑

LFV 的历史具有两个独立的拓扑层（见 `decisions.zh-cn.md §13、§18`）：

- **Snapshot 层**（物理存储）：永远是有向树，每条 Snapshot 恰好有一个 parent 指针，结构永不改变。
- **内容层**（派生视图）：以 object hash 为节点，将 Snapshot parent 关系投影到 object 身份上。单分支 object hash 唯一性不变量保证此层在任意单条分支上无环，整体是 DAG。

### 4.3 可变索引（Mutable Index）

可变索引位于 `index.db`，记录当前工作目录中各文件的工作区状态及分支/标签缓存。它是可重建的当前状态缓存，不属于不可变历史对象。存储对象不可篡改，是仓库的真理之源；可变状态出错可以重建（只要 Object + Snapshot 还在）。

`index.db` 主要包含以下索引表：

| 表 | 说明 | 可重建 |
| ---- | ---- | ---- |
| `file_states` | 核心字段：`fullpath`、`status`（`untracked` / `modified` / `unmodified`）、`file_id`。正常仅登记 LFV 可见、且工作树上当前存在的文件。| ✓ |
| `scan_meta` | 扫描元数据：`last_completed_at`、`.lfvignore` 与 `config.yaml` 的 mtime，供增量扫描失效判定。 | ✓ |
| `branches` | 每个文件的分支指针缓存。真理源为 `.lfv/<file-id>/branches.yaml`。 | ✓ |
| `tags` | 每个文件的标签缓存。真理源为 `.lfv/<file-id>/tags.yaml` 及 `.lfv/trees/tags.yaml`。 | ✓ |
| `tree_file_refs` | tree 维度的反向引用缓存。真理源为 `.lfv/trees/snapshots.log` + `objects/`，`rebuild-index` 时重建；`lfv snap --tree` 时写入。供 `lfv log <file>` 渲染时附加 tree 关联信息。 | ✓ |

> [!note] file_states 说明
> - **正常仅登记 LFV 可见、且工作树上当前存在的文件**（`stat` 成功且不匹配 `.lfvignore` 的路径）；
> - `untracked` 行对应 `config.yaml` 动态 untracked 列表且盘上存在的路径；
> - 盘上已消失则删除该行（`config.yaml` 列表可保留）；
> - **例外**：已跟踪、盘上消失、待 `lfv snap` 记录当前 FS 状态的条目保留为 `modified`，`status` 输出时显示为 `D`（§6.5）。
> - 对已跟踪且盘上仍存在的文件，`fullpath → file_id` 供 CLI 解析路径；盘上消失后仍可用 `file-id` 定位。

## 5. 仓库结构

```
A/                                   # 工作目录
├── docs/note.md                     # 被跟踪文件示例
├── photos/2025/sunset.jpg
└── .lfv/
    ├── config.yaml                  # 仓库级配置（严格 YAML）
    ├── index.db                     # 可重建的状态/分支/标签/tree_file_refs 缓存（SQLite）
    ├── objects/                     # 内容寻址对象存储（File Object + Tree Object）
    │   ├── ab/
    │   │   ├── cdef0123...zstd      # zstd 压缩 File Object
    │   │   └── cdef0123...raw       # 原样 File Object（过小/过大/不可压）
    │   ├── 7f/
    │   │   └── a3bc9d12...raw       # Tree Object（清单 YAML，通常较小 → .raw）
    │   └── ...
    ├── files/                       # 每个跟踪文件的元数据
    │   ├── <file-id>/
    │   │   ├── meta.yaml            # 文件级元数据（创建时间、可选属性）
    │   │   ├── HEAD                 # 当前分支名
    │   │   ├── branches.yaml        # 该文件的分支表
    │   │   ├── tags.yaml            # 该文件的标签表
    │   │   └── snapshots.log        # append-only 的快照记录（JSON Lines）
    │   └── ...
    ├── trees/                       # 全局元数据
    │   ├── snapshots.log            # append-only 的全局快照记录（JSON Lines）
    │   ├── tags.yaml                # tree 标签表
    │   └── HEAD                     # 当前 tree-head，存 snap_<ULID> 或为空
    └── logs/                        # CLI 操作日志（可选，便于调试）
```

### 5.1 对象存储

- 内容寻址：对象身份 = `blake3(原始字节)`；分桶路径中的 hash 前缀来自原始内容，与是否压缩无关；
- 分桶：前 2 个十六进制字符作为目录名，避免单目录文件过多；
- 压缩：对原始字节使用 `zstd`（默认 level **3**）压缩后落盘，扩展名 `.zstd`；读对象时在内存中解压；
- 原样存储（扩展名 `.raw`）：满足以下 **任一** 条件时不压缩：
  - **floor**：`size < min_bytes`（默认 **4 KiB**，4096 字节）——过小，帧开销可能使压缩后更大；
  - **ceiling**：`size >= max_bytes`（默认 **16 MiB**，16777216 字节）——避免大文件整段读入内存时的峰值；
  - **无效压缩**：尝试压缩后 `compressed_len >= original_len`（`reject_if_larger`，默认 **true**）——对已压缩二进制等无收益内容；
- 跨文件共享：相同内容只存一份对象，节省空间。

#### 5.1.1 压缩默认配置（`config.yaml`）

```yaml
compression:
  enabled: true
  algorithm: zstd
  level: 3              # zstd 1..=22
  min_bytes: 4096       # floor: size < min → .raw
  max_bytes: 16777216   # ceiling: size >= max → .raw
  reject_if_larger: true
```

### 5.2 快照记录格式

`snapshots.log` 每行一个 JSON 对象（JSON Lines）：

```json
{
  "id": "snap_01HXYZ...",
  "parent": "snap_01HXYY...",
  "path": "docs/note.md",
  "object": "blake3:abcdef0123...",
  "size": 12345,
  "created_at": "2026-05-17T09:21:33Z",
  "author": "goosy",
  "message": "fix typo in title",
  "tags": [],
  "digest": "blake3:fedcba9876..."
}
```

字段说明：

- `id`：单调可读的快照标识符，采用 [ULID](https://github.com/ulid/spec) 加 `snap_` 前缀。ULID 自带时间戳前缀 + 随机后缀，便于在 `log` 中按时间排序，也便于人在终端粘贴。
- `parent`：父快照 id；分支首个快照为 `null`。
- `path`：**此次快照发生时该文件在工作树上的相对路径**。改名/移动事件就体现为本字段与 `parent.path` 不同；若文件当前不存在，则保留此前的路径（便于 `log` 阅读）。
- `object`：所引用 Object 的 blake3 hash（带 `blake3:` 前缀以便日后切换算法）；**当文件当前不存在时，此字段为 `null`**。
- `size`：对应 Object 的字节数；`object = null` 时为 `0`。
- `digest`：**对本条快照记录除 `digest` 字段以外的所有字段做规范化序列化后的 `blake3` 哈希**，用于防篡改校验。`lfv verify` 会逐行重算并比对。
- 其余字段含义如名所示。

快照不记录所属分支——分支是外部的可变指针（`branches.yaml`），快照只是历史事实。一条快照可以同时属于多个分支的历史路径，删除某个分支不影响快照本身。

设计为 append-only，便于增量备份和审计。

#### 5.2.1 事件类型的推导

LFV 不在 Snapshot 中显式存"事件类型"字段，而是根据 `(parent, path, object)` 三元组与父快照的差异 **派生**：

| 父 path | 父 object | 当前 path | 当前 object | 事件类型 | `log` 中渲染 |
| ---- | ---- | ---- | ---- | ---- | ---- |
| —（首条） | — | P | O | **add**（首次 snap） | `A  P` |
| P0 | O0 | P0 | O1 | **modify** | `M  P0` |
| P0 | O0 | P1 | O0 | **rename** | `R  P0 -> P1` |
| P0 | O0 | P1 | O1 | **rename + modify** | `R+M  P0 -> P1` |
| P0 | O0 | P0 | **null** | **delete** | `D  P0` |
| P0 | **null** | P1 | O1 | **revive** | `+  P1` |

> `track` 动作只登记 `file-id` 与跟踪标志，**不产生 Snapshot**。Snapshot 链的第一条由用户显式执行 `lfv snap` 产生，事件类型推导为 `add`（`parent=null`，`object≠null`）。`track` 与首次 `snap` 是两个独立步骤，自动 track 亦然。

> 不显式存事件类型的好处：每条 Snapshot 都是完整的"在此刻文件是怎样的"快照，添加新事件类型时无需引入新字段或迁移旧记录；坏处：事件类型必须由读侧推导，所以渲染逻辑要单元测试覆盖全部组合。

> **关于"为什么 Snapshot id 用 ULID 而不是内容 hash"**：
> git 用 commit 的内容 hash 作为 id，天然防篡改但 id 不可预读、不可按时间排序。LFV 选择 ULID 作为 id，因为：
> 1. 单文件场景下，用户经常需要在 `log` 输出里凭肉眼按时间挑选快照；ULID 的可读性远好于纯 hash；
> 2. 防篡改职责交给单独的 `digest` 字段，语义清晰，便于 `verify` 命令针对性校验；
> 3. id 与 digest 解耦，将来想增删元数据字段时不会引起 id 漂移。

### 5.3 Tree Snapshot 格式

**Tree Snapshot**（`.lfv/trees/snapshots.log`，每行一个 JSON 对象）：

```json
{
  "id": "snap_01HABC...",
  "parent": "snap_01HABZ...",
  "path": null,
  "object": "blake3:7fa3bc9d...",
  "created_at": "2026-05-17T10:00:00Z",
  "author": "goosy",
  "message": "第三章完成",
  "tags": [],
  "digest": "blake3:fedcba98..."
}
```

与 File Snapshot 的区别：
- `path` 字段为空（tree 不需要定位路径）。
- `object` 不可为 null（tree 快照始终为对象——"整个工作目录为空"时清单为零字节，blake3 hash 照常计算，是合法的 tree-id）。

### 5.4 Tree Object 格式

**Tree Object**（存放于 `objects/`，内容寻址）采用 YAML 块序列格式，每条目一行：

```yaml
- "docs/ch1.md": blake3:abc123...
- "docs/ch2.md": blake3:def456...
- "docs/ch3.md": blake3:ghi789...
```

#### 5.4.1 格式规范

- **结构**：YAML 块序列（block sequence），每行一个单键映射（single-key mapping）。
- **键**（path）：始终加双引号。路径仅含 Unix 分隔符 `/`，不含 `\`；路径内唯一需转义的字符是 `"`（转义为 `\"`），实际路径中几乎不出现。
- **值**（hash）：`blake3:` 前缀 + 小写十六进制，不加引号（不含任何 YAML 特殊字符）。
- **排序**：条目按 path 字典序（UTF-8 字节序）严格升序排列，不允许重复路径。
- **空清单**：工作目录下无有效跟踪文件时，Tree Object 内容为空字节串（零字节），其 blake3 hash 照常计算，是合法的 tree-id。
- **无多余内容**：不含注释、不含空行、不含 BOM、文件末尾恰好一个换行符（`\n`）；空清单则为零字节，没有换行符。

#### 5.4.2 规范化与 tree-id

blake3 直接对上述字节串摘要，得到该 Tree Object 的身份（tree-id）。由于格式完全确定（引号、排序、换行），同样的工作目录内容必然产生逐字节相同的序列化结果，进而产生相同的 tree-id——无需额外的规范化步骤，存储格式即规范化格式。

两次内容完全相同的工作目录状态——无论何时、以何种方式达到——产生相同的 Tree Object hash，`objects/` 里只有一份条目。

#### 5.4.3 存储与兼容性

Tree Object 通常很小（每个跟踪文件一行），低于 `min_bytes` 压缩下限，以 `.raw` 存储。

该格式是合法的 YAML 子集，外部程序可直接用标准 YAML 解析器读取，无需了解 LFV 内部约定。LFV 自身解析时，也可用一行正则 `/^- "(.+)": (\S+)$/` 逐行提取，不依赖完整解析器。

**对比两个 Tree Snapshot**：逐条比对各自 Tree Object 清单。`path` 相同且 `object` hash 相同——未变；`path` 相同但 `object` hash 不同——已修改；仅出现在其中一个清单里——新增或删除。

### 5.5 `index.db` 表结构

`index.db` 只存放**可重建**的缓存数据，真理源均在文件系统（Snapshot 链、yaml 文件）中。

```sql
-- 工作树文件状态缓存
CREATE TABLE file_states (
    fullpath   TEXT PRIMARY KEY,  -- 相对仓库根的路径
    file_id    TEXT,              -- f_<ULID>；untracked 时为 null
    status     TEXT NOT NULL      -- 'untracked' | 'modified' | 'unmodified'
);

-- 扫描元数据
CREATE TABLE scan_meta (
    key        TEXT PRIMARY KEY,  -- 'last_completed_at' | 'lfvignore_mtime' | 'config_mtime'
    value      TEXT NOT NULL
);

-- 分支指针缓存（真理源：.lfv/<file-id>/branches.yaml）
CREATE TABLE branches (
    file_id    TEXT NOT NULL,     -- f_<ULID>
    name       TEXT NOT NULL,     -- 分支名
    snap_id    TEXT NOT NULL,     -- 该分支 HEAD 的 snap_<ULID>
    PRIMARY KEY (file_id, name)
);

-- 文件标签缓存（真理源：.lfv/<file-id>/tags.yaml）
CREATE TABLE tags (
    file_id    TEXT NOT NULL,     -- f_<ULID>
    name       TEXT NOT NULL,     -- 标签名
    snap_id    TEXT NOT NULL,     -- snap_<ULID>
    PRIMARY KEY (file_id, name)
);

-- tree 维度反向引用缓存（真理源：tree/snapshots.log + objects/）
CREATE TABLE tree_file_refs (
    tree_snap_id  TEXT NOT NULL,  -- snap_<ULID>
    file_id       TEXT NOT NULL,  -- f_<ULID>
    file_object   TEXT NOT NULL,  -- blake3:...
    PRIMARY KEY (tree_snap_id, file_id)
);
```

**`tree_file_refs` 的写入与重建**：`lfv snap --tree` 创建 Tree Snapshot 时展开 Tree Object 清单，逐 file 插入一行；`rebuild-index` 时通过遍历所有 Tree Snapshot 重建。供 `lfv log <file>` 渲染时附加 tree 关联信息：

```
snap_01HXYZ  M  docs/note.md   "add chapter 2"
             └─ tree: "第三章完成" [t:v1.0]
snap_01HWWW  M  docs/note.md   "fix typo"
             └─ tree: "日常归档 2026-05-30"
```

### 5.6 不可重建的状态文件

以下文件记录**用户意志**，无法从 Snapshot 链机械推导，`rebuild-index` 时不覆盖：

**`.lfv/<file-id>/HEAD`**

每个跟踪文件一个，内容为当前所在分支名（纯文本，一行）：

```
main
```

LFV 不支持 detached HEAD——`rewind` 强制新建分支，所以 HEAD 始终指向一个具名分支，不会是裸 snap\_id。

**`.lfv/trees/HEAD`**

tree 层当前头部，内容为 `snap_<ULID>` 或空（仓库尚无 Tree Snapshot 时）：

```
snap_01HABC...
```

### 5.7 分支与标签文件（真理源）

以下 yaml 文件是分支/标签的真理源，`index.db` 中的 `branches`、`tags` 表是其可重建缓存。

**`.lfv/<file-id>/branches.yaml`**

key = 分支名，value = 该分支当前 HEAD 的 `snap_<ULID>`：

```yaml
main: snap_01HXYZ...
rewind/01HXY0/1: snap_01HABC...
```

- 分支首次创建时写入；`lfv snap` 在当前分支上产生新快照后更新对应 value。
- `lfv branch-delete` 删除对应 key；快照本身不受影响（append-only）。

**`.lfv/<file-id>/tags.yaml`**

key = 标签名，value = `snap_<ULID>`，只增不改（标签不可复用）：

```yaml
v1.0: snap_01HXYZ...
stable: snap_01HWWW...
```

**`.lfv/trees/tags.yaml`**

tree 层标签表，key = `"t:<name>"`，value = `snap_<ULID>`，标签全局唯一，不允许复用：

```yaml
t:v1.0: snap_01HABC...
t:release: snap_01HZZZ...
```

用户输入的标签名不允许包含 `:`（命名空间隔离，见 `decisions.zh-cn.md §20`）；`t:` 前缀由 LFV 自动添加。

## 6. 文件生命周期事件

本节描述所有改变文件跟踪状态的动作——包括 LFV 自动执行的扫描响应，以及用户显式执行的命令操作。

### 6.1 跟踪策略：默认全跟踪

LFV 采用"**默认全跟踪**"策略：工作目录下的文件，正常情况下都应当处于跟踪状态。为此，LFV 在相关命令执行前对工作树做**惰性扫描**（算法见 §6.8），自动响应 OS 层的新建和删除事件，无需后台守护进程。

### 6.2 自动 track（新建文件）

扫描时发现工作树中存在一个 **LFV 可见**、且未进入状态表的路径，即：

- 若匹配 `.lfvignore` → **忽略**（不登记，§6.6）；
- 若在 `config.yaml` 动态 untracked 列表中 → 登记为 `untracked`（§6.6）；
- 否则 → 视为「OS 新建」，自动 `track`，分配 `file-id`，状态 `modified`

**不会立即追加 Snapshot**，由后续 `lfv snap` 落盘首条 Snapshot。

自动 track 的触发时机：`lfv status`（扫描时即时执行）、`lfv track`（无参跟踪时执行）、`lfv snap`（无参批量拍照前执行）。

**取消 track**：`lfv untrack <file>` 写入 `config.yaml` 动态 untracked 列表；路径在盘上存在时同步登记状态表 `untracked`。已有历史者可保证不再产生新 Snapshot；无历史的 auto-track 文件 untrack 后只删除跟踪缓存，以保证不追加 Snapshot。

路径在 config untracked 且盘上存在时，扫描时不会 auto-track；盘上消失则删除状态表行（§6.8），文件再现时按 §6.2 重新登记 `untracked`。

**同名文件的友好提示**：若某路径曾有一个已删除的 `file-id`（即该路径最新 Snapshot 的 `object = null`），自动 track 在同一路径上分配了新的 `file-id` 后，`lfv status` 会在该文件条目下附加提示：

```
  M   f_01HA7EEE...   docs/old-note.md  [newly tracked]
                      note: this path previously existed as f_01HA7BCD (deleted)
                      to continue its history instead, run:
                        lfv relink f_01HA7EEE --onto f_01HA7BCD
```

此提示仅在自动 track 产生新 `file-id` 时出现；一旦新 `file-id` 落下首条 Snapshot，它就是独立的文件，提示消失。

对于有历史的 untracked 文件，由于它的最新 Snapshot 仍指向某个 Object，当再次人工 `lfv track <file>` 时，LFV 会先从 `config.yaml` 中的动态 untracked 列表中移除，再按照原 `file-id` 进入跟踪，并重新在状态表中对应文件更新为 tracked 状态以缓存。注意，这里与 delete 文件后 `lfv track <file>` 的算法不一样，后者一定默认产生新的 `file-id`，除非人为地 `lfv revive`。

### 6.3 rename / move（lfv mv）

**`lfv mv` 只做路径操作**：`<src>` 和 `<dst>` 均只允许路径，不接受 file-id。用于文件在磁盘上仍存在（或刚被 OS 移动）时的改名/移动。不允许 `<dst>` 为 file-id 的主要原因是：它会造成心智混乱——用户可能误以为保留的 file-id 是 `<dst>` 那一侧。若需要把历史续接到另一个 file-id，请使用 `lfv relink`（见 §6.4）。

- `lfv mv <src> <dst>`：把 src 对应的跟踪文件迁移到 dst 路径。
  - 若工作树上 src 路径仍存在且 dst 不存在，CLI 会先把文件移到 dst，再追加快照（原子语义）。
  - 若 src 路径已不存在（OS/编辑器先动了），dst 路径上已有该文件，则 `lfv mv` 只做"登记快照"，不再动盘。
  - 新 Snapshot 的 `object` 由 dst 当前内容的 hash 决定：若与父快照相同则呈现为纯 `R`，不同则呈现为 `R+M`。
  - `message` 默认为 `rename: <old-path> -> <new-path>`。
- **自动识别 vs. 手动登记**：
  - 自动识别仅在内容哈希 **完全一致** 时生效（`rename.autodetect`，默认开启）。一旦命中，`lfv status` 直接以 `R` 行显示并自动配对。
  - 若 OS 移动文件 **并且** 内容也变了，自动识别会失败——`lfv status` 会同时显示一条 `D` 行（旧 file-id 在原路径消失）和一条 `A` 行（新路径上的新 file-id）。用户检视后用 `lfv relink <new-file> --onto <old-file>` 显式声明历史续接。
  - 自动识别也可被关闭（适用于成批改名 + 编辑的场景，避免误配对）。
- 历史回溯：`lfv log <file>` 会以"`R` 旧路径 -> 新路径"形式渲染 rename 事件，与 `A`(add) / `M`(modify) / `D`(delete) / `R+M`(rename+modify) 并列（见 §5.2.1）。

### 6.4 历史续接（lfv relink）

`lfv relink <f_src> --onto <f_dst>` 用于将 f_src 的历史在 f_dst 的历史结尾重新接续，同时盘上 f_src 对应文件将以后与 <f_dst> 绑定。如果 f_src 还没有历史，则 f_dst 的历史维护不变。

它的一个典型应用场景是：扫描器无法自动配对的 rename——用户在 OS 层修改了路径和内容，LFV 将其识别为独立的 `D`（f_dst）+ `A`（f_src）两个事件，由用户手动声明"f_src 是 f_dst 的延续"。

- **f_src**：状态 `A`/`M`，盘上存在的活文件（新 file-id，尚在跟踪中）
- **f_dst**：状态 `D`，已从磁盘消失的文件（旧 file-id，待续接）
- **最终保留的 file-id 是 `--onto` 的那一侧（f_dst）**，与介词方向一致

执行后 f_src 的当前路径和内容作为 f_dst 历史的下一条 Snapshot 追加；f_src 的 `file_states` 跟踪行取消，不再产生新快照。**f_src 的 `files/<f_src>/` 目录及 `snapshots.log` 完整保留**（append-only，不删除），f_src 成为"已退役"的 file-id，`lfv log f_src` 仍然有效。

**情况一：f_src 尚无快照历史**

f_src 只有跟踪记录，未产生任何 Snapshot。执行后：将 f_src 的当前路径的文件使用 f_dst 这个 file-id。事件类型按 §5.2.1 由 f_dst 末尾快照与当前的 path/object 对比推导（通常为 `R+M` 或 `R`）。

**情况二：f_src 已有快照历史**

f_src 已经产生了若干 Snapshot（链为 `snap_A1 -> snap_A2 -> ... -> snap_An`）。执行后，将 f_src 的整条快照链复制并追加到 f_dst 历史末尾：

- 每条快照新生 ULID（`snap_B1 ... snap_Bn`），`digest` 对新字段重算
- `snap_B1.parent` = f_dst 的最新快照；`snap_Bx.parent` = `snap_B(x-1)`
- `snap_B1` 的事件类型由 f_dst 末尾快照与 `snap_B1` 的 path/object 对比推导；`snap_B2` 之后与原 f_src 内部推导结果一致，不受影响
- 接缝处的事件类型可能呈现为跨路径的跳变，这是用户主动声明历史续接的预期代价
- 同样，当前盘上 f_src 对应文件将以后与 <f_dst> 绑定

### 6.5 自动 delete（OS 删除文件）

扫描时发现某个**跟踪文件**的路径在工作树上消失，且没有证据表明是改名（即改名自动识别未能配对），视为"OS 删除文件"。LFV **不立即**追加 Snapshot，而是将其状态标记为 `modified`；因为当前路径在 FS 上不存在，`lfv status` 会显示为 `D`。

后继执行 `lfv snap` 命令时，会落地状态的更新。所有后继处理的说明见 §6.6。

这样设计的好处：`lfv snap` 之前，`D` 文件仍在跟踪列表中，用户还有机会通过 `lfv mv f_old <new-path>` 将其识别为改名，避免误删。

### 6.6 lfv delete

`lfv delete <file>`：在状态表中将该文件设置为 modified，同时删除该文件。后继 `lfv snap` 时，LFV 会参照目标文件的当前 FS 存在状态：若文件不存在，则追加 `object = null` 的 Snapshot，并从状态表的当前路径索引中移除对应的 `file-id`。

此后该 file-id 不再能通过当前路径解析，但最后的路径信息保留在该 Snapshot 中，`lfv list --deleted` 仍可按路径显示。

文件 **历史完整保留**：仍可 `lfv log` / `lfv show` / `lfv diff` 查询；想"复活"用 `lfv revive`（详见 §7.2）或直接 `lfv rewind <snap>`，二者都会自动新建分支（不破坏既有删除事件）。

### 6.7 `.lfvignore` 与 `config.yaml`

默认全跟踪会把临时文件、构建产物等也纳入扫描；`.lfvignore` 与 `config.yaml` 提供了不同级别的「排除列表」，它们的职责不同：

| | `.lfvignore` | `config.yaml`（动态 untracked） |
| ---- | ---- | ---- |
| 性质 | 静态、仓库级、**最高优先级** | 用户通过 `lfv untrack` / `lfv track` 维护的动态策略 |
| 状态表 | **不进入** `file_states` | 盘上**存在**时登记 `untracked`；盘上消失则**删除**状态表行（config 列表保留） |
| 工作树扫描 | **不可见**：剪枝跳过，不参与 OS 新建/删除检测 | **可见**：参与 §6.8；仅对盘上存在的路径维护状态表行 |
| `lfv track <file>` | 匹配则**报错**，须改 `.lfvignore` | 从 untracked 列表移除并进入跟踪 |
| `lfv untrack <file>` | 匹配则**报错** | 写入 config；盘上存在则登记 `untracked` |
| `lfv status --include-untracked` | **不会出现** | **唯一来源**（状态表中的 `untracked` 行，均源于 config） |

补充约定：

- `.lfv/` 目录本身永远隐式视为 `.lfvignore` 规则，不进入状态表；
- 匹配 `.lfvignore` 的路径，对 LFV 而言等同于**不存在**：不 auto-track、不 untrack、不列入 `--include-untracked`；
- `rebuild-index` 时，`untracked` 行 = `config.yaml` 动态列表 ∩ LFV 可见路径 ∩ **盘上存在**的路径。

### 6.8 工作树扫描（惰性扫描）

若干命令（`lfv status`、`lfv track`、`lfv snap` 等）执行前会触发工作树扫描，对比磁盘与 `index.db`，更新 `file_states` 并驱动 §6.2–§6.6 的自动动作。

#### 6.8.1 扫描动作

先定义几个概念：

- 动态未跟踪：一个路径在 `config.yaml` 动态 untracked 列表中。
- 可跟踪候选：一个路径不在 `config.yaml` 动态 untracked 列表中。
- 状态更新：
  - 如果 LFV 可见且为动态未跟踪：设置为 `untracked` 状态；
  - 如果 LFV 可见且已有 tracked 状态行：对比 mtime/size → 对比 hash → 设置为 `modified` 或 `unmodified`，判断链条不一定要执行完才能结果；
  - 如果 LFV 可见、未登记且为可跟踪候选：执行 auto-track（§6.2）；
  - LFV 不可见：删除状态表中对应记录，保持 `config.yaml` 与历史 Snapshot 不变，不产生 `D`；
- 动态 untracked：在 `config.yaml` 动态 untracked 列表中的路径。

| 层级 | 对象 | 扫描动作 |
| ---- | ---- | ---- |
| **A. 已登记路径** | `file_states` 中已有 `fullpath` 的行 | 对每行先依据 `.lfvignore` mtime 决定是否判断 LFV 可见，不可见则删除状态表记录，可见则状态更新。成本 O(已登记路径数)。 |
| **B. 发现新路径** | 遍历中见到的、尚未登记的 LFV 可见路径 | 动态未跟踪文件设置为 `untracked`；可跟踪候选执行 auto-track（§6.2）。 |

以上扫描动作都会更新状态表。此外，`lfv track`、`lfv untrack` 也会更新 `config.yaml` 与 `file_states`（`untracked`）。

#### 6.8.2 增量扫描与失效

`scan_meta` 记录上次扫描完成时刻与规则文件 mtime。默认**增量**扫描，避免每次命令都全量递归整个工作树：

- **失效**（任一成立则层级 B 做受控全量遍历）：
  - `scan_meta` 为空（首次扫描或 `rebuild-index` 之后）；
  - `.lfvignore` 的 mtime 晚于 `scan_meta` 中记录值（遍历/剪枝边界变化）；
  - `config.yaml` 的 mtime 晚于记录值（按动态 untracked 列表与LFV可见路径**对账** `untracked` 行存在则登记，不存在则删除）；
  - 用户执行 `lfv status --refresh`（或等价强制刷新）。
- **否则（增量）**：
  - 对**目录**：若目录 mtime ≤ `last_completed_at` 且该目录已在扫描登记中 → 不 descend；与 `.lfvignore` **目录剪枝**叠加（剪枝目录对 LFV 不可见）；
  - 对**文件**：若已在 `file_states` 且文件 mtime ≤ `last_completed_at` → 跳过层级 B 的「是否新路径」判定（层级 A 仍 `stat`）。
- **扫描结束**：更新 `last_completed_at`（建议取扫描开始时刻）及 `.lfvignore` / `config.yaml` 的 mtime。

> [!note] 注意
> 依赖 mtime 的增量策略在拷贝未保留时间戳、或文件系统秒级精度不足时可能漏扫；用 `--refresh` 兜底。

#### 6.8.3 触发时机

| 命令 / 场景 | 扫描 |
| ---- | ---- |
| `lfv status` | 默认增量扫描（§6.8.2） |
| `lfv status --refresh` | 强制失效后扫描 |
| `lfv track`（无参）、`lfv snap`（无参） | 扫描后再批量 track / snap |
| `lfv status --include-untracked` | **不**改变扫描；仅多打印状态表中的 `untracked` 行（§7.3.2） |

## 7. CLI 命令

> 约定：`<file>` 指工作目录内某个文件的相对或绝对路径；CLI 在内部统一规范为相对于仓库根的相对路径。

### 7.1 仓库管理

| 命令 | 说明 |
| ---- | ---- |
| `lfv init` | 在当前目录创建 `.lfv/`。已存在则报错。 |
| `lfv config <key> [value]` | 读取/写入仓库配置（如 `user.name`）。 |

### 7.2 文件跟踪

| 命令 | 说明 |
| ---- | ---- |
| `lfv track [<file>]` | 把某文件加入跟踪。指定 `<file>` 时，若路径匹配 `.lfvignore` 则报错；否则从 `config.yaml` 中 untracked 列表移除；若该路径此前是 untrack 而非 delete，则复用原 `file-id`，并将状态表更新为 `modified`（无历史快照时等待首次 `lfv snap`）。无 file 参数时，自动扫描所有可跟踪且尚未进入状态表的文件并加入跟踪。 |
| `lfv untrack <file>` | 停止跟踪：LFV不可见则报错；可见则更新至 `config.yaml` 动态 untracked 列表；若盘上存在则状态表登记 `untracked`，否则仅 config；历史保留，可 `lfv track` / `lfv revive`。 |
| `lfv mv <old> <new>` | 把 `<old>` 路径对应的跟踪文件迁移到 `<new>` 路径。`<old>` 和 `<new>` 均只接受路径，不接受 file-id。是否在工作树上执行实际的文件移动，由 §6.3 规定。 |
| `lfv relink <f_src> --onto <f_dst>` | 把 f_src 的历史续接到 f_dst 上，声明"f_src 是 f_dst 的延续"。f_src 的快照（若有）以新生 ULID 追加到 f_dst 历史末尾；f_src 的 `file_states` 跟踪行取消，但其 `snapshots.log` 完整保留。详见 §6.4。 |
| `lfv delete <file>` | 在状态表中将该文件设置为 modified，同时删除该文件。后续 `lfv snap` 时检测到它是 modified 且文件不存在时，追加一条 `object = null` 的 Snapshot，并移除出 `config.yaml` 中 untracked 列表。更新状态表的缓存状态。它的历史完整保留，随时可 `lfv revive`。 |
| `lfv revive <ref>` | 复活已删除的文件。`<ref>` 可以是 `file-id`、最近已知路径或某条 Snapshot id。自动新建分支（`revive/<...>`），从所选快照恢复内容到工作树。 |
| `lfv list [--deleted]` | 列出所有被跟踪文件，每个记录包括路径、当前分支、最新快照ID及摘要。默认仅列活跃文件，`--deleted` 同时列出最新 Snapshot 的 `object = null` 的文件。 |

### 7.3 状态与快照

| 命令 | 说明 |
| ---- | ---- |
| `lfv status [<file>]` | **省略 `<file>` 时列出所有 `modified` 的跟踪文件**；执行前按 §6.8 做惰性扫描。默认输出仅含 tracked 变更，每行带 `file-id`（`f_*`）。`--include-untracked` 见 §7.3.2。`--refresh` 强制全量刷新扫描缓存。指定 `<file>` 时仅显示该文件。 |
| `lfv snap [<file>] [-m <msg>]` | 为某文件创建新快照。**省略 `<file>` 时，自动对所有 `modified` 状态的跟踪文件批量拍照**。若工作区内容未变化则拒绝（除非 `--allow-empty`）。`--tree` 参数见 §7.3.3。 |
| `lfv log <file>` | 列出该文件的快照历史，附带 tree 关联信息（来自 `tree_file_refs` 表）。支持 `--branch <name>`、`--graph`、`--limit N`。 |
| `lfv log --tree` | 列出 tree 历史视图：沿 Tree Snapshot 链，每个节点显示 message、标签、时间戳。 |
| `lfv show <file> <snap>` | 输出某个快照的元数据；`--content` 输出内容；`--out <path>` 导出。 |

说明：
- `lfv status` 命令对于 `modified` 文件，被删除的文件显示为 `D`，路径与上个快照不符的显示为 `R`，没有快照的显示为 `A`，其它的显示为 `M`，对内容和路径都有改变的，显示为 `R+M`。
- `lfv snap` 时，会对每个目标 modified 文件检查当前 FS 状态：文件存在则写入/复用 Object 并追加内容 Snapshot，文件不存在则追加 `object = null` 的 Snapshot；`unmodified` 文件跳过。执行前按 §6.8 做惰性扫描；若指定 `<file>`，仅处理该文件。

#### 7.3.1 `lfv status` 输出格式（跟踪文件）

`lfv status` 输出形如：

```
$ lfv status --include-untracked

Tracked files (changes):
  M   f_01HA7BCD...   docs/note.md
                      content changed (12.4 KB -> 12.7 KB)
  R   f_01HA7XYZ...   docs/photo.jpg -> docs/2026/photo.jpg
                      auto-detected (identical content hash)
  D   f_01HA7DEF...   docs/old-note.md
                      file missing on disk; will be deleted on next `lfv snap`
                      or run `lfv mv f_01HA7DEF <new-path>` if it was moved
  M   f_01HA7EEE...   docs/old-note.md  [newly tracked]
                      note: this path previously existed as f_01HA7BCD (deleted)
                      to continue its history instead, run:
                        lfv relink f_01HA7EEE --onto f_01HA7BCD

Untracked (config.yaml):
  ~   drafts/local.md                  (2.1 KB)
```

**标志位**：

- `A`=add（新跟踪，尚无快照）
- `M`=modified（含首次 track 尚未 snap 的新文件）
- `R`=rename（自动识别出的待落盘改名；显式 `lfv mv` 会直接追加 Snapshot）
- `D`=suspected delete（**已跟踪**且 `modified` 的文件盘上消失；下次 `lfv snap` 会按当前 FS 状态追加 `object = null` 的 Snapshot）
- `~`=动态 untracked（`status = untracked`，config 策略 + 盘上文件存在）

**`file-id` 列**：

显示 `f_*`（ULID 加 `f_` 前缀），默认缩写为前 10 字符，`--long` 显示全长。该值在 track 时分配，贯穿文件整个生命周期，是 CLI 上唯一稳定的引用 token。

**任何接受 `<file>` 的命令通常同时接受路径或 `f_*` 作为参数**：

- `lfv relink f_01HA7EEE --onto f_01HA7BCD` —— 将 f_01HA7EEE（新 file-id，盘上存在）的历史续接到 f_01HA7BCD（旧 file-id，已消失）上；f_01HA7BCD 继续作为活跃 file-id，f_01HA7EEE 退役保留。
- `lfv mv docs/old-note.md docs/notes/new.md` —— 路径改名。
- `lfv delete f_01HA7DEF` —— 即便文件已不在工作树，仍可用 file-id 显式登记删除。

#### 7.3.2 `--include-untracked`（展示动态 untracked）

与 §6.8 工作树扫描是**独立**主题：该标志**只改变输出**，不触发额外全树遍历。

- **唯一数据源**：`file_states.status = untracked` 的行（均已通过扫描确认盘上存在；§6.8.1）。
- LFV可见，即不出现 `.lfvignore` 匹配路径（§6.6），`.lfvignore` 文件为静态排除规则， `lfv status` 不会列出LFV不可见文件。
- 在跟踪文件区块后追加 **Untracked** 区块，以 `~` 列出（无 `file-id`），`stat` 取大小；盘上已消失的路径无状态表行，故无 `~` 行。

输出示例见 §7.3.1；`~` 含义见该节标志位说明。

#### 7.3.3 `lfv snap --tree`：创建 Tree Snapshot

```bash
lfv snap --tree -m "第三章完成"
lfv snap --tree --tag v1.0 -m "第一版完成"   # 创建时同步打标签
```

执行前提：所有被跟踪文件均无 modified 状态（否则报错，提示先执行 `lfv snap`）。

执行流程：
1. 读取所有跟踪文件当前 HEAD 下有效的 file-object-hash，构建 Tree Object 清单（按路径排序）。
2. 规范化序列化后计算 blake3 hash，得到 tree-id；若 `objects/` 中已有该对象则复用，否则写入。
3. 向 `.lfv/trees/snapshots.log` append 一条 Tree Snapshot 记录，parent 指向当前 tree HEAD（若为空则为 null）。
4. 更新 `.lfv/trees/HEAD` 为新 Tree Snapshot 的 id。
5. 若指定 `--tag`，向 `.lfv/trees/tags.yaml` 写入标签。
6. 展开 Tree Object 清单，向 `index.db` 的 `tree_file_refs` 表逐 file 插入一行。

### 7.4 对比

| 命令 | 说明 |
| ---- | ---- |
| `lfv diff <file>` | 工作区 vs. 最新快照。 |
| `lfv diff <file> <snap>` | 工作区 vs. 指定快照。 |
| `lfv diff <file> <snapA> <snapB>` | 两快照之间。 |

文本文件使用基于行的 diff（默认上下文 3 行）；二进制文件仅显示元数据差异（大小、hash）。

### 7.5 回溯与分支

| 命令 | 说明 |
| ---- | ---- |
| `lfv rewind <file> <snap>` | 把工作区文件内容恢复到指定快照。**自动新建分支**（命名规则 `rewind/<snap-short>/<n>`），并将 HEAD 切到新分支。 |
| `lfv rewind <t:tag|snap-id>` | 把工作区还原到指定 Tree Snapshot 的状态。更新 `.lfv/trees/HEAD`；对每个 file 采用 FF 优先策略（详见 §7.5.1）。 |
| `lfv branches <file>` | 列出该文件的全部分支。 |
| `lfv switch <file> <branch>` | 切换该文件的当前分支（同时把工作区内容更新为该分支头部快照）。仅针对 file 分支，不操作 tree。 |
| `lfv branch-rename <file> <old> <new>` | 重命名分支。 |
| `lfv branch-delete <file> <branch>` | 删除分支（仅删除指针，对象保留以防共享）。 |

#### 7.5.1 `lfv rewind t:<tag|snap-id>` 的执行流程

**前置校验**：检查工作区是否存在 modified 但尚无 file-object 的文件（新建、修改或删除但未 `lfv snap`）。若存在则拒绝执行：

```
error: the following files have unsaved changes (no file-object yet):
  M  docs/draft.md
run `lfv snap` first, or discard changes manually.
```

**执行流程**：

1. 将 `.lfv/trees/HEAD` 更新为目标 Tree Snapshot 的 snap id。
2. 读取目标 tree-snapshot 的 Tree Object，得到 `{path, file-object-hash}` 清单。
3. 按以下三类分别处理所有 file：

**分类 A：在清单中的 file（需还原内容）**，对每个 file 以 `target_hash` 为目标，执行 FF 优先 rewind：
  - **FF 路径**：若某分支的 HEAD object hash == `target_hash` → 优先选当前分支；无则选最近创建的匹配分支 → 直接 `switch`，不新建分支。提示：`[FF] docs/note.md → branch main`
  - **rewind 路径**：否则 → 执行标准 rewind，自动新建分支。提示：`[rewind] docs/note.md → new branch rewind/01HXYZ/1`

**分类 B：不在清单中、但仓库已有 file-object 的 file**（tree-snapshot 后新增并已快照过的 file）：
  → 仅删除工作区文件，不动 `config.yaml`。后续扫描（§6.5）自动标记为 `D`。提示：`[deleted] docs/new-chapter.md`

**分类 C：在清单中、但当前已 untracked 或 deleted 的历史 file**：
  → 仅将文件字节写回工作区，不修改任何状态表或 config。后续扫描接管：
  - 原 file-id 为 deleted：走 §6.2 同名文件友好提示，用户可 `lfv relink` 续接历史。
  - 原 file-id 为 untracked：扫描后显示 `~`，用户自行决定是否 `lfv track`。
  提示：`[restored] docs/old-chapter.md`

4. 输出汇总。

### 7.6 标签

| 命令 | 说明 |
| ---- | ---- |
| `lfv tag <file> <snap> <name>` | 给某文件快照打标签。 |
| `lfv tags <file>` | 列出该文件的所有标签。 |
| `lfv tag-delete <file> <name>` | 删除文件标签。 |
| `lfv tag --tree <snap> <name>` | 给 Tree Snapshot 打标签（存储为 `t:<name>`）。 |
| `lfv tags --tree` | 列出所有 tree 标签。 |
| `lfv tag-delete --tree <name>` | 删除 tree 标签。 |

用户输入的标签名不允许包含 `:`（命名空间隔离）。

### 7.7 维护

| 命令 | 说明 |
| ---- | ---- |
| `lfv gc` | 回收未被任何快照引用的对象。 |
| `lfv verify` | 校验对象存储的完整性（重算 hash 比对）。 |
| `lfv export <file> [--format zip|tar] -o <out>` | 导出某文件的全部历史为独立归档，便于迁移。 |

## 8. 典型工作流

### 8.1 第一次使用

```bash
cd /path/to/A
lfv init
# 可选：创建 .lfvignore 静态禁止跟踪的路径
echo "node_modules/" >> .lfvignore
echo "*.tmp" >> .lfvignore

lfv status
# -> 惰性扫描（§6.8）：可跟踪的新文件被自动 track（打上跟踪标志，无 Snapshot）
#    所有文件均显示为 `A` 标志，在状态表中为 modified 状态

lfv snap -m "initial snapshot"   # 对所有 modified 文件批量拍照，产生首条 Snapshot
```

### 8.2 日常编辑

```bash
# 编辑 docs/note.md ...
lfv status docs/note.md         # 查看是否有未保存变更
lfv snap docs/note.md -m "add chapter 2 outline"
lfv log docs/note.md
```

### 8.3 对比

```bash
lfv diff docs/note.md
lfv diff docs/note.md snap_01HXYZ
```

### 8.4 回溯到旧版本（自动分支）

```bash
lfv log docs/note.md
lfv rewind docs/note.md snap_01HXY0
# -> 自动新建分支 rewind/01HXY0/1，并把工作区切到该分支
# 之后的 snap 都会落到该新分支上，原 main 分支历史完整保留
```

### 8.5 改名 / 移动

```bash
# 场景 A：通过 LFV 改名（推荐，原子）
lfv mv docs/note.md docs/notes/2026-05/note.md
# -> 追加一条 rename 快照，path 由旧变新，object 不变

# 场景 B：先用 OS 改了名，未改内容 —— 自动识别即可
mv docs/note.md docs/notes/2026-05/note.md
lfv status
#   R   f_01HA7BCD   docs/note.md -> docs/notes/2026-05/note.md
#                    auto-detected (identical content hash)
lfv snap                        # 一并落盘所有已识别变更

# 场景 C：先用 OS 改了名，又改了内容 —— 自动识别失败，手动续接历史
mv docs/note.md docs/notes/2026-05/note-v2.md
$EDITOR docs/notes/2026-05/note-v2.md
lfv status
#   D   f_01HA7BCD   docs/note.md
#                    file missing on disk; possibly moved
#   M   f_01HA7EEE   docs/notes/2026-05/note-v2.md  [newly tracked]
lfv relink f_01HA7EEE --onto f_01HA7BCD
# -> f_01HA7EEE 的内容（当前路径 + object）作为 f_01HA7BCD 历史的下一条 Snapshot 追加
# -> 事件类型由 f_01HA7BCD 末尾快照与新 Snapshot 对比推导（通常为 R+M）
# -> f_01HA7EEE 退役（file_states 取消，snapshots.log 保留）
```

### 8.6 删除与复活

```bash
lfv delete docs/old-note.md     # 删除工作树文件，并将状态表标记为 modified
lfv snap -m "remove old note"   # 追加 object = null 的 Snapshot（也可留给下次无参 lfv snap 批量处理）
lfv log docs/old-note.md        # 历史依然可查
lfv revive docs/old-note.md     # 自动新建 revive/<...> 分支并恢复内容到工作树
```

### 8.7 同名新文件接续旧历史

```bash
# 场景：docs/old-note.md 曾被删除（该 file-id 的最新 Snapshot 为 object = null），
#        现在在同一路径新建了一个文件

lfv status
#   M   f_01HA7EEE...   docs/old-note.md  [newly tracked]
#                       note: this path previously existed as f_01HA7BCD (deleted)
#                       to continue its history instead, run:
#                         lfv relink f_01HA7EEE --onto f_01HA7BCD

# 选择 A：新文件就是新文件，与旧历史无关，直接 snap
lfv snap docs/old-note.md -m "new document"

# 选择 B：新文件是旧文件的延续，接续旧历史
lfv relink f_01HA7EEE --onto f_01HA7BCD
# -> f_01HA7EEE 的内容作为 f_01HA7BCD 历史的下一条 Snapshot 追加（revive 事件）
# -> f_01HA7EEE 退役（file_states 取消，snapshots.log 保留，不留活跃痕迹）
```

### 8.8 全局快照（tree 层）

```bash
# 编辑完当前版本的所有文件后，先确保全部 snap
lfv snap -m "finish chapter 3"

# 创建 tree-snapshot，标记里程碑
lfv snap --tree -m "第三章完成"
lfv snap --tree --tag v1.0 -m "第一版完成"   # 同时打标签

# 查看 tree 历史
lfv log --tree
# snap_01HABC  2026-05-30 10:00  "第一版完成" [t:v1.0]
# snap_01HXYZ  2026-05-17 09:21  "第三章完成"

# 回到某个 tree-snapshot（工作区内容整体还原）
lfv rewind t:v1.0
lfv rewind t:snap_01HXYZ
```

### 8.9 跨设备同步

直接通过 NAS / 网盘把整个工作目录（含 `.lfv`）拷贝/同步到另一设备即可。LFV 本身 **不解决并发写入冲突**——同步工具的责任是确保不会同时修改 `.lfv`。

## 9. 技术选型

- **语言**：Rust 2024 edition。
- **CLI 框架**：`clap` v4，使用 derive 风格定义命令树。
- **错误处理**：`thiserror`（库级别定义错误类型） + `anyhow`（CLI 顶层收尾）。
- **哈希**：`blake3`（速度快，足够强）。
- **压缩**：`zstd` level 3；默认 `min_bytes` 4 KiB、`max_bytes` 16 MiB、`reject_if_larger: true`（详见 §5.1）。
- **元数据序列化**：`serde` + 严格 YAML（人类可读的配置/元数据） + `serde_json`（snapshots.log 行格式）。`.lfv` 下所有配置与元数据文件统一使用 `.yaml` 后缀；严格 YAML 指 LFV 只写入和接受一个受限子集：映射、序列、字符串、数字、布尔值和 null，不依赖锚点、别名、复杂 tag 或隐式类型推断。
- **索引存储**：`rusqlite`（嵌入式 SQLite，单文件 `index.db`）。
- **文本 diff**：`similar`（行/词级 diff，输出 unified 格式）。
- **时间**：`time` 或 `jiff`（待评估，倾向 `jiff` 以获得更现代的 API）。
- **日志**：`tracing` + `tracing-subscriber`，CLI 通过 `-v/-vv` 控制级别。
- **测试**：`assert_cmd` + `predicates` + `tempfile` 做集成测试；单元测试就近放在 mod 中。

> 选型原则：优先使用已被 Rust 生态广泛验证的库，避免引入不维护的 crate。所有依赖在 1.0 之前每季度审视一次。

## 10. 工程描述

工程目录：

```
src/
├── main.rs                # CLI 入口，仅做参数解析 + 调度
├── cli/                   # clap 命令定义与各子命令的执行入口
│   ├── mod.rs
│   ├── init.rs
│   ├── track.rs
│   ├── snap.rs
│   ├── log.rs
│   ├── diff.rs
│   ├── rewind.rs
│   └── ...
├── repo/                  # 仓库抽象：打开、关闭、路径解析、配置
│   ├── mod.rs
│   ├── config.rs
│   └── layout.rs
├── index/                 # SQLite 索引访问层
│   └── mod.rs
├── object/                # 对象存储：写入、读取、压缩、GC
│   └── mod.rs
├── snapshot/              # 快照实体、序列化、append 写入 snapshots.log
│   └── mod.rs
├── branch/                # 分支管理
│   └── mod.rs
├── tag/                   # 标签管理
│   └── mod.rs
├── diff/                  # diff 算法封装
│   └── mod.rs
└── util/                  # 通用工具：路径、时间、字符串
    └── mod.rs
```

测试目录：

```
tests/
├── cli_init.rs
├── cli_track.rs
├── cli_snap.rs
├── cli_rewind.rs
└── ...
```

构建与发布：

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

## 11. 路线图（粗略）

- **v0.1** ：`init` / `track` / `snap` / `log` / `status` / `show`。
- **v0.2** ：`diff` / `rewind` / `branches` / `switch`。
- **v0.3** ：`tag` 系列、`export`、`gc`、`verify`。
- **v0.4** ：tree 层（`snap --tree` / `log --tree` / `rewind t:` / `tag --tree`）、性能优化。
- **v1.0** ：稳定 CLI 语义，文档完整，跨平台 CI 通过。
