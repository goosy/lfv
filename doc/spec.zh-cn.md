# LFV 轻量文件中心版本控制

> 本文件是 LFV 项目的中文版规范（spec）：定义要建什么、为什么、行为与约束。英文版见 `spec.md`。
> 实施设计（怎么建：存储结构、内部机制、工程结构）见 `design.zh-cn.md`；本文以 `design §x` 引用其章节。

---

## 1. 项目背景与动机

`git` 是非常优秀的版本控制工具，但它的设计目标是 **以仓库为单位** 的版本管理，最适合一组逻辑相关的源代码集合：一次提交可以横跨多个文件，分支记录的是整个工作树的整体演化。

然而日常工作中还存在另一类典型场景，它具有以下特征：

- 每个文件 **逻辑独立**，相互之间没有耦合；
- 每个文件 **自有一条演化历史**，与其它文件无关；
- 用户关心的是 **某一个具体文件** 在时间轴上的演化、对比、回退。
- **"一次性提交多文件"**的语义需求并不是必须的；

典型例子：

- **NAS 文件同步系统**：每个备份文件都有自己的历史，但不需要跨文件的"原子提交"。
- **一对一笔记仓库**：每一篇笔记独立演化，A 笔记的修改不应"污染"B 笔记的历史。
- **设计稿/合同/单文档档案库**：每个文档单独跟踪版本。
- **配置文件目录**：每个配置文件独立演进，互不影响。

目前业界对这类场景的处理普遍不令人满意：

- **简单备份**：仅复制旧版本，无法直观看到每个版本的目的；
- **保留 N 个历史版本**：缺乏语义信息，无法 diff、无法标注；
- **整目录 git 仓库**：把"无关文件"的演化混在一起，历史线被淹没，违背"以文件为中心"的心智模型；
- **云盘历史版本**：通常不可离线使用，也不支持分支、对比、可移植。

## 2. 项目目标

LFV (Lightweight File Versioning) 的目标：

1. 提供 `lfv` CLI，使用户能像使用 `git` 那样直观地对某个目录(即仓库repo)下的所有文件：
   - 选择性地跟踪某个文件；
   - 为该文件创建带注释的快照；
   - 查看该文件的历史快照列表；
   - 对比任意两个快照（或快照与当前内容）；
   - 回溯（rewind）到某个历史快照——回溯时 **自动建立新分支保存旧路径**，绝不破坏既有历史。
2. 每个文件拥有 **独立的**：分支历史、分支集合、标签集合。
3. 文件的每个分支必须是一条单向**不可分叉**的路径，不能有中途的分叉与合并。
4. 提供良好的可移植性：`.lfv` 目录可以随工作目录一起拷贝/同步，迁移后行为一致。
5. 数据存储紧凑：通过内容哈希去重和压缩，避免简单备份导致的存储膨胀。
6. 分布式协作：实现与本机或网络的仓库进行协调同步，实现基本的 push / pull / remote / merge 等协同语义。

为了保持"轻量"，下列内容 **不在** 本项目范围内：

- **分布式通讯协议**：LFV 不约束网络仓库的底层协议，仅规定通过远程引用获取 `.lfv` 目录的接口。实现方可自行扩展驱动，以支持 `ssh:uname@host:~`、`https://other.dns/somerepo.lfv` `https://webdav.host/somerepo` 等地址格式。
- **合并范围限制**：`merge` / `rebase` 仅针对单个文件的分支，不跨文件协调。
- **暂存操作**：一律"工作区即快照应用部分"，需要分离版本，可对快照进行回溯修改。
- **替代 git**：在源代码工程场景下应继续使用 git；LFV 只服务"文件中心"场景。
- **图形界面**：只提供 CLI，GUI 应当另起项目来完成。

## 3. 术语规范

### 3.1 基本术语

| 概念 | 英文 | 说明 |
| ---- | ---- | ---- |
| 仓库 | Repository | 工作目录下的 `.lfv` 目录，承载所有被跟踪文件的元数据与对象存储。 |
| 工作树 | Working Tree | 工作目录本身（除 `.lfv` 外），用户实际操作的文件位于此处。 |
| 跟踪文件 | Tracked File | 由 `lfv track` 加入仓库管理的文件。内部以 `file-id`（ULID）作为稳定身份，与路径完全解耦。 |
| 文件状态 | File Status | 对已纳入 LFV 视野的路径，状态为 `unmodified`、`modified` 或 `untracked`。（见 design §4.3） |
| 对象 | Object | 内容寻址的文件内容或全局引用存储单元；按内容哈希去重，不同文件可能共享同一对象。 |
| 快照 | Snapshot | 某个文件或全局文件在某一时刻的拍照：内容指针 + 路径 + 元数据（消息、时间戳、作者、父快照）。|
| 分支 | Branch | 某个跟踪文件下的一条快照链；默认分支 `main`。每个文件的分支命名空间彼此独立。 |
| 头部 | HEAD | 某个跟踪文件当前所在的分支名与最新快照指针。 |
| 标签 | Tag | 对某个快照的可读命名（可选），用于稳定地引用某个版本。 |
| 动作 | Action | 改变文件状态的操作。显式动作：`track`、`snap`、`untrack`；隐式动作：`modify`（用户编辑文件）、`auto-track` / `auto-delete`（LFV 扫描时自动响应 OS 事件，见 design §4.3）。 |
| LFV 可见 | — | 在 `.lfvignore` 匹配目录剪枝后，工作目录中剩余的文件。 |

### 3.2 存储对象：File Object、Tree Object、File Snapshot、Tree Snapshot

LFV 的存储对象有两种 **完全不可变、内容寻址** 的类型 —— File Object、Tree Object —— 组成，均存放于 `.lfv/objects/`，按内容 hash 去重。它们共同点是，写一次，永不修改。可通过 `lfv verify` 重算 hash 校验完整性。

LFV 的快照是 **append-only 的事件记录层**，以 ULID 寻址，记录"某个时刻发生了什么"。它也分两种类型：File Snapshot、Tree Snapshot，结构高度对称，它们分别用于**两个独立视图**。

所有快照 append-only，不会重写已有快照。

> [!note] 快照与对象的不可变性原则
> LFV 的 append-only 保证是"**可遍历范围内**"的快照不可修改。不可达（悬空）的快照不在此约束之内，可以安全物理删除。File Object、Tree Object 作为内容寻址单元，其不变性更强——写入即永不变更。这一原则是 `lfv gc --purge` 等删除操作的安全基础，详见 §5.9。

> [!note] File Object 与历史的关系
> File Object 本身只是内容单元，不携带任何历史信息——它没有父子关系，不知道自己属于哪条分支，也不知道自己在哪个时间点产生。历史信息完全由 Snapshot 链承载。
> 凡是需要历史上下文的操作（`lfv log`、`lfv merge`、`lfv rebase`），其参数必须是 `<FS-ish>`（快照 id，或带文件上下文的分支名 / 标签名），不能是纯粹的 file-object hash——后者仅标识"内容是什么"，不标识"历史在哪里"。
> `<FO-ish>` 的各种形式（见 §4 导语）均可在只需要内容的场合使用。

#### 3.2.1 File Object

File Object 是单个文件原始字节的内容寻址存储单元。它的身份标识为文件原始内容的 `blake3` 哈希——不同路径或历史的文件，若内容相同，共享同一个 File Object。File Object 完全不可变：写入一次，永不修改。存储细节（压缩、分桶、去重）见 design §3.2。

`file-id` 是某个跟踪文件在仓库内部的**稳定身份标识**：

- **首次跟踪时**分配一个独立的 ULID。
- 与文件路径**无任何派生关系**，不会因改名、移动而变化；文件删除后 `file-id` 依然有效。
- 文件的"当前路径"由可变索引（§3.4）维护，并随 Snapshot 链上的事件持续派生更新——`index.db` 中的可变状态均可从 Object、Snapshot 链与 `.lfv` 下的元数据文件重建；HEAD、分支表、标签表与 `config.yaml` 记录的是用户意志，不能从快照链推导，各自即是真理源。

#### 3.2.2 Tree Object

对用户而言，这是仓库快照，仓库快照术语用于用户界面，本规范统一为 Tree Object。

该对象是存储工作目录某一时刻所有有效跟踪文件的内容清单：按路径字典序排列的 `{path, file-object-hash}` 条目列表，形成规范化清单。tree-id 为该清单字节串的 `blake3()`，内容寻址。两次工作目录状态完全相同，产生同一 Tree Object hash，`objects/` 里只存一份。工作目录下有效跟踪文件为空时，即清单为空——这仍是合法的 Tree Object。

Tree Object 与 File Object 共用同一 `objects/` 目录，身份计算、去重、存储机制完全一致（见 design §3.2）。Tree Object 通常体积很小，一般以原样存储。

#### 3.2.3 File Snapshot

每条 File Snapshot 是某个跟踪文件在某一时刻的完整事件记录，核心字段：

- `id`：`snap:<ULID>`，时间有序、全局唯一，不可重写。
- `parent`：父快照 id；分支首条为 `null`。
- `path`：此次快照时文件在工作树上的相对路径（路径是快照的字段，不是文件的身份）。
- `object`：所引用 File Object 的 blake3 hash；文件不存在时为 `null`（delete 事件）。
- `digest`：对本条记录除 `digest` 外所有字段做规范化序列化后的 blake3 hash，供 `lfv verify` 防篡改校验。

此外其它信息字段，消息、作者、时间等，参看 design §3.3。快照不记录标签——标签是外部可变指针（design §3.8）。

File Snapshot 有以下特点：

- **不显式存事件类型**
- **不记录所属分支**，快照只记录历史事件。
- 整个文件生命周期就是 File Snapshot 链。
- "重命名/移动"、"新增"、"内容变更"、"删除"仅在用户界面上有意义，靠对父快照字段比对即可区分。由读侧根据 `(parent.path, parent.object, path, object)` 四元组推导（add / modify / rename / rename+modify / delete / revive），详见 design §3.3.1。
- 每条 File Snapshot 只有一个 parent 指针，整体构成**有向树（森林）**：分支出去即独立演化，不存在拓扑意义上的合并点。

#### 3.2.4 Tree Snapshot

每条 Tree Snapshot 是整个工作目录的里程碑记录，主要存储以下内容：

- `id`：`snap:<ULID>` 仅 append，不重写
- `object`：tree-object 指针，指向一个 Tree Object（永不为 null）
- `parent`：父快照 id
- `digest`：对本条记录除 `digest` 外所有字段做规范化序列化后的 blake3 hash，供 `lfv verify` 防篡改校验。

其它字段如消息、作者、时间等请参看 design §3.4。

Tree Snapshot 与 File Snapshot 结构对称，但字段有以下差异：

- `path` 字段为空（tree 不需要定位路径）。
- `object` 不可为 null（tree 快照始终指向一个 Tree Object）。

树面（tree plane）没有分支概念，因此也没有 `branches.yaml`。

全局快照对用户来说可选：仓库可以没有任何 Tree Snapshot。

Tree Snapshot 同样只有一个 parent 指针，构成有向树，所有 Tree Snapshot 存放于同一个 append-only 日志中（存储路径与格式见 design §3.4）。

从一个 Tree Snapshot 出发，可通过 Tree Object 清单里的 file-object-hash 定位到某个文件在那个时刻的内容；进而在该文件的 Snapshot 链中查找 object hash 匹配的条目，即可找到对应的 File Snapshot（分支对象唯一性不变量保证匹配的快照在一条分支上至多构成一段连续 run，取其中 path 相同的最后一条）。清单本身不含 file-id，"哪个文件"由 tree 反向引用缓存给出（design §3.6）。

#### 3.2.5 拓扑面

LFV 的历史具有两个独立的拓扑面：

- **快照拓扑面**（物理存储）：永远是有向树，每条 Snapshot 恰好有一个 parent 指针，结构永不改变。
- **内容拓扑面**（派生视图）：以 object hash 为节点，将 Snapshot parent 关系投影到 object 身份上形成的 DAG 图（连续指向同一 object 的快照折叠为同一节点）。分支对象唯一性保证此面在任意单条分支上不会有回环。

即，object 与 snapshot 是解耦关系，前者只有内容，后者描述拓扑，一个 object 可以有多个 snapshot 时间线经过。

### 3.3 快照同的判定和同的祖先

在 LFV 中，不同的快照可能指向同一个 File Object，LFV 称这２个快照**同的(co-referent)**，很多 LFV 操作就是建立在`同的`的基础上。

比如变基和合并操作，就需要定位的同的祖先：在两条分支的 Snapshot 链中，分别向上回溯，找到各自历史上都曾出现过且`同的`的最近节点或节点对：

- **base-snap-ours**：本分支历史中指向共同 file-object 的那条 Snapshot
- **base-snap-theirs**：目标分支历史中指向同一 file-object 的那条 Snapshot
- **base-object**：两者共同指向的 file-object（内容相同，只有一份）

两分支的同的祖先 snap 可能是同一个，也可能是不同（当两条分支各自独立经历了相同内容时），所以base-snap-ours与base-snap-theirs往往并不是同一个快照。基于这个同的祖先的合并操作，特别是同的祖先非同一个快照的更普遍情况下的合并，LFV 称之为4路合并(4-way merge)。

### 3.4 可变索引（Mutable Index）

LFV 维护一个可重建的**可变索引**（Mutable Index）作为工作区状态缓存，避免每次命令都全量扫描文件系统；推荐实现为嵌入式数据库。存储对象（Object）、Snapshot 链与 `.lfv` 下的元数据文件（HEAD、分支表、标签表、`config.yaml`）共同构成仓库的真理源，索引损坏时可随时从真理源重建（`lfv rebuild-index`，§4.8）。详见 design §3.6。

### 3.5 元数据文件的 YAML 约束

本节约束 `.lfv/` 下**可变的用户意志/元数据文件**——`config.yaml`、`meta.yaml`、`branches.yaml`、`tags.yaml`、`trees/tags.yaml`、`REPLAY.yaml`（详见 design §3.7、§3.8）。`HEAD`、`trees/HEAD` 是纯文本，不受本节约束。Tree Object 虽然也编码为 YAML，但它是内容寻址对象，规范形已由 design §3.5 单独定义，且以字节精确匹配为目的，不适用本节规则。

目标：写出的文件必须是任意标准 YAML 解析器可以正常读取的合法 YAML；同时写法要收紧到可以用一个不依赖通用 YAML 库的极简解析器读写。不论 LFV 自身实现用完整 YAML 库还是自写最简解析器，本节都是磁盘格式的硬性约束，不是解析器的可选行为。

**语义子集**：只使用映射、序列、字符串、数字、布尔值与 null；不使用锚点、别名、复杂 tag 或隐式类型推断。

**书写规约**：

1. 只用 block 风格，不出现 flow 风格（`{...}`/`[...]`）。
2. 缩进固定为每层 2 个空格；序列固定以 `- ` 起行。
3. 固定 schema 的结构（`meta.yaml`、`config.yaml`、`REPLAY.yaml` 的 `Step`）字段按其结构体声明顺序全部输出；`Option` 为空时显式写 `~`，不省略 key。
4. 注释只允许出现在整行行末；因为值要么是裸写的安全值、要么强制加引号（见下），"从第一个未加引号的 `#` 到行尾"永远是安全的注释边界。
5. 文件内不含空行；文件末尾恰好一个换行符。

**字符串按可控程度分三类，处理方式不同**：

- **用户可控字符串**——内容来自用户输入、LFV 不限制其字符集（`RepoPath`、`config.yaml` 的 `user.name`、snapshot 的 `author`）。**一律强制双引号**，引号内按标准 YAML 双引号转义规则书写（等同 `serde_json` 字符串转义：只转义 `"`、`\`、控制字符，非 ASCII 原样 UTF-8）。`RepoPath` 已经禁止 `"`/`\`（§4.2），落到本条规则时天然不需要真正转义；`user.name`/`author` 没有字符限制，需要按此规则完整转义。
- **用户部分可控、由 LFV 批准**——分支名、标签名（`BranchName`/`TagName`）。用户提出名字，但只有通过 LFV 的创建时校验才会真正存在，因此可以对字符集设限，换取继续裸写（不加引号）。具体字符规则见 §4.6，各条约束的由来见 design §2.1.1。
- **LFV 自己控制**——`snap:<ULID>`、`file:<ULID>`、`blake3:<hex>`、RFC 3339 时间戳、`true`/`false`、整数版本号等。字符集由 LFV 自身定义且已知安全，裸写，不加引号、不转义。

## 4. CLI 功能规范

以下参数约定，最终解析为 4 个存储对象之一（File Object、Tree Object、File Snapshot、Tree Snapshot）。分支名与标签名是**每个文件独立的命名空间**（§2 目标 2），因此不能单独出现，必须带文件上下文。

- `<file>` 定位一个跟踪文件（而非它的某个版本），两种写法：
  - 路径：`a`、`./a`、`../a` 相对当前工作目录；以 `/` 开头的 `/a/b` 是仓库内绝对路径（以仓库根为根，不是操作系统绝对路径）。不接受操作系统绝对路径与盘符，各平台写法一致。统一规范化为相对仓库根的路径，解析到仓库之外则报错；盘上已消失的路径按历史中最后拥有该路径且未退役的 file-id 解析（多个候选取其 HEAD 快照 `created_at` 最晚者，仍相同则取 snap-id 最大者）；该路径一旦被新文件占用，就解析到新的活跃 file-id，因此稳定引用已消失的文件应使用 file-id；
  - `<file-id>`，即 `file:<ULID>`。
- `<FS-ish>` 最终解析为 File Snapshot 的参数，包括：
  - `<snap-id>`：某个快照标识 `snap:<ULID>`，全局唯一，自带文件归属；
  - `<file>`：该文件当前分支的 HEAD 快照；
  - `<branch>:<file>`：该文件某分支的 HEAD 快照；
  - `<tag>:<file>`：该文件某标签所指的快照。
  写法与 git 的 `<rev>:<path>` 一致。路径、分支名、标签名都不含 `:`，分支名与标签名也不能是命名空间保留字（`file`、`snap`、`tree`、`work`、`blake3`），因此 `:` 的切分没有歧义；文件名不受保留字限制。
  在已带 `<file>` 参数的命令中（如 `lfv rewind <file> <FS-ish>`），`<FS-ish>` 可省略 `:<file>`，直接写分支名、标签名或 snap-id；解析顺序为 snap-id → 分支名 → 标签名（同一文件内分支名与标签名不得重名，见 §4.6，故此顺序只用于与 snap-id 区分）。
- `<FO-ish>` 最终解析为 File Object 的参数，包括：
  - 任何 `<FS-ish>`：解析为该快照的 `object`；
  - `work:<path>`：工作区当前内容，当成一个未保存的特殊 File Object。
- `<TO-ish>` 最终解析为 Tree Object 的参数，包括：
  - `<tree-id>`：Tree Object 的内容 hash（`blake3:...`）；
  - `tree:<tag>`：树标签所指树快照对应的 Tree Object；
  - `<snap-id>`：某个树快照对应的 Tree Object（该 snap-id 须属于树面）。
- `<TS-ish>` 最终解析为 Tree Snapshot 的参数，包括：
  - `<snap-id>`：某个树快照标识（须属于树面，snap-id 全局唯一，归属由索引确定）；
  - `tree:<tag>`：树标签所指的树快照。

### 4.1 仓库管理

| 命令 | 说明 |
| ---- | ---- |
| `lfv init` | 在当前目录创建 `.lfv/`。已存在则报错。 |
| `lfv config <key> [value]` | 读取/写入仓库配置（如 `user.name`）。 |

### 4.2 文件跟踪

| 命令 | 说明 |
| ---- | ---- |
| `lfv track [<file>]` | 把某文件加入跟踪。指定 `<file>` 时，若路径匹配 `.lfvignore` 则报错；否则从 `config.yaml` 中 untracked 列表移除；若该路径此前是 untrack 而非 delete，则复用原 `file-id`，并将状态表更新为 `modified`（无历史快照时等待首次 `lfv snap`）。无 file 参数时，自动扫描所有可跟踪且尚未进入状态表的文件并加入跟踪。 |
| `lfv untrack <file>` | 停止跟踪：LFV不可见则报错；可见则更新至 `config.yaml` 动态 untracked 列表；若盘上存在则状态表登记 `untracked`（保留其 file-id 以便 `lfv track` 复用），否则仅 config；历史保留，可 `lfv track` / `lfv revive`。 |
| `lfv mv <old-file> <new-file>` | 把 `<old-file>` 路径对应的跟踪文件迁移到 `<new-file>` 路径。`<old-file>` 接受路径或 file-id（file-id 必定对应一个路径）；`<new-file>` 只接受路径，不接受 file-id。若移动后的内容违反分支对象唯一性（design §4.13）则拒绝整条命令，此时盘上文件尚未移动。是否在工作树上执行实际的文件移动，由 design §4.6 规定。 |
| `lfv relink <src-file-id> --onto <dst-file-id>` | 将 src-file-id 当前分支的历史续接到 dst-file-id 当前分支末尾，盘上文件改由 dst-file-id 标识，src-file-id 退役。**仅针对两个 file-id 各自当前分支**，不涉及其他分支，不做 4-way merge 内容合并。专为误产生新文件的场景设计，不应作为日常命令。详见 design §4.7。 |
| `lfv delete <file>` | 在状态表中将该文件设置为 modified，同时删除该文件。后续 `lfv snap` 时检测到它是 modified 且文件不存在时，追加一条 `object = null` 的 Snapshot，并移除出 `config.yaml` 中 untracked 列表。更新状态表的缓存状态。它的历史完整保留，随时可 `lfv revive`。 |
| `lfv revive <file> [<FS-ish>]` | 复活已删除的文件。默认恢复点为当前分支最后一条 `object != null` 的快照，可用 `<FS-ish>` 指定其它快照。实现为 rewind（§4.5）：分支名不变，HEAD 移到恢复点，含删除事件的原 HEAD 由自动新建的 `revive/<anchor-short>/<n>` 分支保留；内容写回工作树的最后已知路径。同名新文件要接续旧历史不走 revive，用 `lfv relink`（§5.7）。 |
| `lfv list [--deleted] [--all]` | 列出所有被跟踪文件，每个记录包括路径、当前分支、最新快照ID及摘要。默认仅列活跃文件，`--deleted` 同时列出最新 Snapshot 的 `object = null` 的文件；`--all` 再加上已退役（relink 的 src）与已 untrack 但有历史的 file-id。 |

**路径规则**：为保证任一平台上记录的历史都能在其它平台上还原，可跟踪路径统一采用 Windows 文件名规则（Linux、macOS 禁止的字符是其子集）：

- 必须是合法 UTF-8；路径以 Unicode NFC 形式记录与比较，磁盘上以其它规范形（如 macOS 常见的 NFD）存储的同名文件视为同一路径，由 LFV 自动识别；
- 不含控制字符，不含 `<` `>` `:` `"` `\` `|` `?` `*`（`/` 只作分隔符）；
- 每个路径分量非空，且不以 `.` 或空格结尾；
- 每个路径分量去掉第一个 `.` 及其后内容后，不得（不区分大小写）是 Windows 保留名：`CON`、`PRN`、`AUX`、`NUL`、`COM1`–`COM9`、`LPT1`–`LPT9`、`COM¹`–`COM³`、`LPT¹`–`LPT³`。

不满足者：自动 track 不登记，只给出警告（扫描照常继续）并提示加入 `.lfvignore`；显式 `lfv track <file>` 报错。路径以 `/` 为分隔符存储并逐字节比较（大小写不敏感的文件系统上，仅改大小写也视为改名）。符号链接不跟随、不跟踪；空目录不跟踪。

**路径输入**：CLI 在解析路径参数时也接受 `\` 作为分隔符（它不可能出现在文件名里），但 LFV 的输出与提示一律只使用 `/`，不鼓励使用 `\`。在 MinGW / Git Bash 中，以 `/` 开头的参数会被 MSYS 改写成 Windows 路径，仓库内绝对路径须写成 `//docs/a.md`（LFV 把开头连续的多个 `/` 视同一个）。

**跨平台差异的处理**：以下问题由用户处理，LFV 只负责检测和报错，检测不到时照常运行。

- **仅大小写不同的路径**：在大小写不敏感的文件系统上（如 Windows 默认），若仓库记录中存在两个仅大小写不同的活跃路径，或一次操作会同时写出这样两个路径，LFV 报错并提示用户为相关目录开启大小写敏感（Windows：`fsutil.exe file setCaseSensitiveInfo <dir> enable`）；不存在这种情况时照常运行。
- **Unicode 规范化冲突**：同一目录下两个磁盘文件名仅规范形不同（规范化为 NFC 后相同，通常只在 Linux 上出现）时，LFV 报错，由用户改名或加入 `.lfvignore`。
- **路径长度**：Windows 默认的路径长度限制由用户处理（启用长路径支持或缩短路径）；LFV 只在文件系统操作因路径过长失败时报错说明原因。

### 4.3 状态与快照

| 命令 | 说明 |
| ---- | ---- |
| `lfv status [<file>]` | **省略 `<file>` 时列出所有 `modified` 的跟踪文件**；执行前按 design §4.3 做惰性扫描。默认输出仅含 tracked 变更，每行带 `file-id`（`file:*`）。`--include-untracked` 见 §4.3.2。`--refresh` 强制全量刷新扫描缓存。指定 `<file>` 时仅显示该文件。 |
| `lfv snap [<file>] [-m <msg>]` | 为某文件创建新快照。**省略 `<file>` 时，自动对所有 `modified` 状态的跟踪文件批量拍照**。若工作区内容与路径均与 HEAD 快照相同则拒绝。若违反分支对象唯一性（design §4.13），则拒绝创建快照并提示用户执行 `lfv rewind`。`--snap-all` 与省略 `<file>` 等价，只是让「批量、共用同一条 message」的意图在命令行上可见，不能与 `--tree` 同时使用；`--tree` 参数见 §4.3.3。 |
| `lfv log <FS-ish>` | 列出快照所在分支的历史，附带 tree 关联信息（来自 tree 反向引用缓存）。`<FS-ish>` 为 `<file>` 时显示其当前分支；为 `<branch>:<file>` 时显示该分支；为 snap-id 时显示该快照所在分支——优先当前分支，否则按分支名排序取第一条含它的分支。`--all` 显示该文件所有分支的历史；`--graph` 以 ASCII 图形式渲染分支拓扑；`--limit N` 限制条数。 |
| `lfv log --tree` | 列出树面（tree plane）历史视图：沿 Tree Snapshot 链，每个节点显示 message、标签、时间戳。 |
| `lfv show <FS-ish>` | 输出该快照的元数据；`--content` 同时输出对象内容；`--out <path>` 把内容导出到文件。 |
| `lfv show <FO-ish>` | 不带 `--content`/`--out` 时，输出该 File Object 的身份信息：hash、原始字节数，以及仓库内所有引用该 hash 的快照（`snap-id` + 各自的 `path`）。File Object 是内容寻址、与具体文件无关的存储单元（§3.2.1），同一内容可能被多个不同 file-id、多个分支的快照共同引用，因此这里的快照列表不限于 `<FO-ish>` 解析时经过的那个文件。带 `--content` / `--out` 时只输出对象内容，不输出上述信息。 |

说明：
- `lfv status` 命令对于 `modified` 文件，被删除的文件显示为 `D`，路径与上个快照不符的显示为 `R`，没有快照的显示为 `A`，其它的显示为 `M`，对内容和路径都有改变的，显示为 `R+M`。
- `lfv snap` 时，会对每个目标 modified 文件检查当前 FS 状态：文件存在则写入/复用 Object 并追加内容 Snapshot，文件不存在则追加 `object = null` 的 Snapshot；`unmodified` 文件跳过。执行前按 design §4.3 做惰性扫描；若指定 `<file>`，仅处理该文件（待落盘的自动识别改名，旧路径与新路径都解析到同一文件）。
- 文件尚无任何快照（`A`）且已从盘上消失时，`lfv snap` 不产生快照，只撤销跟踪。
- 批量 `lfv snap` 逐文件独立处理，单个文件失败（如环回）不影响其它文件。

#### 4.3.1 `lfv status` 输出格式（跟踪文件）

`lfv status` 输出形如：

```
$ lfv status --include-untracked

Tracked files (changes):
  M   file:01HA7BCD...   docs/note.md
                      content changed (12.4 KB -> 12.7 KB)
  R   file:01HA7ACE...   docs/photo.jpg -> docs/2026/photo.jpg
                      auto-detected (identical content hash)
  D   file:01HA7DEF...   docs/removed_note.md
                      file missing on disk; will be deleted on next `lfv snap`
                      or run `lfv mv file:01HA7DEF <new-path>` if it was moved
  A   file:01HA7EEE...   docs/old-note.md  [newly tracked]
                      note: this path previously existed as file:023BHCA1 (deleted)
                      to continue its history instead, run:
                        lfv relink file:01HA7EEE --onto file:023BHCA1

Untracked (config.yaml):
  ~   drafts/local.md                  (2.1 KB)
```

**标志位**：

- `A`=add（跟踪但尚无快照；含"该路径此前存在过一个已删除的 file-id"的提示场景，见下方示例）
- `M`=modified（已有快照，内容或路径与 HEAD 不同）
- `R`=rename（自动识别出的待落盘改名；显式 `lfv mv` 会直接追加 Snapshot）
- `D`=suspected delete（**已跟踪**且 `modified` 的文件盘上消失；下次 `lfv snap` 会按当前 FS 状态追加 `object = null` 的 Snapshot）
- `~`=动态 untracked（`status = untracked`，config 策略 + 盘上文件存在）

**`file-id` 列**：

显示 `file:*`（ULID 加 `file:` 前缀），默认缩写为前缀加 ULID 前 8 位（如 `file:01HA7BCD`），`--long` 显示全长。该值在 track 时分配，贯穿文件整个生命周期，是 CLI 上唯一稳定的引用 token。

**任何接受 `<file>` 的命令通常同时接受路径或 `file:*` 作为参数**：

- `lfv relink file:01HA7EEE --onto file:01HA7BCD` —— 将 file:01HA7EEE（新 file-id，盘上存在）的历史续接到 file:01HA7BCD（旧 file-id，已消失）上；file:01HA7BCD 继续作为活跃 file-id，file:01HA7EEE 退役保留。
- `lfv mv docs/old-note.md docs/notes/new.md` —— 路径改名。
- `lfv delete file:01HA7DEF` —— 即便文件已不在工作树，仍可用 file-id 显式登记删除。

#### 4.3.2 `--include-untracked`（展示动态 untracked）

与工作树扫描（design §4.3）是**独立**主题：该标志**只改变输出**，不触发额外全树遍历。

- **唯一数据源**：状态表中的 `untracked` 行（均已通过扫描确认盘上存在）。
- LFV可见，即不出现 `.lfvignore` 匹配路径（design §4.2），`.lfvignore` 文件为静态排除规则， `lfv status` 不会列出LFV不可见文件。
- 在跟踪文件区块后追加 **Untracked** 区块，以 `~` 列出（无 `file-id`），`stat` 取大小；盘上已消失的路径无状态表行，故无 `~` 行。

输出示例见 §4.3.1；`~` 含义见该节标志位说明。

#### 4.3.3 `lfv snap --tree`：创建 Tree Snapshot

```bash
lfv snap --tree -m "第三章完成"
lfv snap --tree --tag v1.0 -m "第一版完成"   # 创建时同步打标签
lfv snap --snap-all -m "第一版完成"   # 用同一 message 逐个 snap 所有 modified 文件
```

**前置条件**：所有被跟踪文件均无 `modified` 状态；否则报错，提示先执行 `lfv snap`。

可以用 `--snap-all` 参数先清除 `modified`：LFV 以同一 `<msg>` 对每个 `modified` 文件执行一次 `lfv snap`（等价于用户手工逐个执行，各文件历史上各得一条独立快照）。任一文件 snap 失败（如环回）不影响其它文件 snap。必须先清除 modified ，再建树快照。 --snap-all 与 --tree 不能同时进行。

**效果**：将当前工作目录所有跟踪文件的内容（各自 HEAD 处的 file-object）捕获为一个内容寻址的全局快照，追加到 Tree Snapshot 链并推进 tree HEAD。若指定 `--tag`，同时写入标签。内部执行步骤见 design §4.11。

### 4.4 对比

工作区当前内容可作为一个未保存的特殊 `File Object`，用 `work:<path>` 字面量引用（§4 导语）。

| 命令 | 说明 |
| ---- | ---- |
| `lfv diff <FO-ish>` | 指定 FO-ish 所指向的 File Object vs. 工作区当前内容。它相当于省略了第二参数 `work:<path>`，路径由第一个参数所属文件的当前路径推断。裸路径作为 `<FO-ish>` 解析为该文件当前 HEAD 的 object，所以 `lfv diff docs/note.md` 即 HEAD vs. 工作区。 |
| `lfv diff <FO-ish-A> <FO-ish-B>` | 两个 File Object 之间的对比。（既可以是不同文件，也可以是同文件不同版本） |

文本文件使用基于行的 diff（默认上下文 3 行）；二进制文件仅显示元数据差异（大小、hash）。

### 4.5 回溯与分支

| 命令 | 说明 |
| ---- | ---- |
| `lfv rewind <file> <FS-ish>` | 把工作区文件内容恢复到 `<FS-ish>` 所指向的快照对应的 File Object（只恢复内容，不改路径；HEAD 快照路径与盘上路径不同时，下次 `lfv snap` 记为 `R`）。**分支名不变**：先自动新建保留分支 `rewind/<anchor-short>/<n>` 指向原 HEAD，再把当前分支 HEAD 移到目标快照。文件处于 `modified` 状态时拒绝执行（先 `lfv snap`，或自行丢弃改动），环回模式除外。用于解决环回事件时，用户传入被匹配的祖先快照本身，LFV 按 design §4.13 的规则接续。 |
| `lfv rewind <TS-ish>` | 把工作区还原到指定 Tree Snapshot 的状态。更新 `.lfv/trees/HEAD`；对每个 file 采用 FF 优先策略（详见 §4.5.1）。 |
| `lfv branches <file>` | 列出该文件的全部分支。 |
| `lfv switch <file> <branch>` | 切换该文件的当前分支（同时把工作区内容更新为该分支头部快照）。仅针对文件面，不操作树面。文件处于 `modified` 状态时拒绝。目标分支 HEAD 为 `object = null`（该分支上文件已删除）时，从工作树删除该文件；该路径此后仍可解析（回退到历史中最后拥有它且未退役的 file-id），但一旦被新文件占用就解析到新 file-id，因此稳定引用它应使用 file-id。 |
| `lfv branch-rename <file> <old> <new>` | 重命名分支。`<new>` 须满足 §4.6 的命名规则，且在该文件内不与现有分支名、标签名重复；重命名当前分支时同步改写该文件的 `HEAD`。 |
| `lfv branch-delete <file> <branch>` | 删除分支（仅删除指针，快照和对象保留，以便共享和找回）。拒绝删除当前 HEAD 所在的分支，需先 `lfv switch` 到其它分支。 |

**保留分支命名规则**：回溯类操作自动新建的保留分支统一命名为 `<kind>/<anchor-short>/<n>`。`kind ∈ {rewind, detour, rebase, revive}` 标明来源操作（分别对应本节 rewind、环回处理 design §4.13、§4.7.1、§4.2 revive）；`anchor-short` 为被保留的旧 HEAD 快照 ULID 的末 8 位；`n` 从 1 起递增以避免重名。树面没有分支，`lfv rewind <TS-ish>` 为离开的 tree HEAD 自动创建的标签 `tree:detour/<anchor-short>/<n>` 沿用同一命名规则，其 `anchor-short` 取离开的 tree HEAD 快照 ULID 的末 8 位。

#### 4.5.1 `lfv rewind <TS-ish>` 的执行流程

**前置校验**：检查工作区是否存在任何 `modified` 状态的跟踪文件（含 `A` / `M` / `R` / `D`）。若存在则拒绝执行：

```
error: the following files have unsaved changes:
  M  docs/draft.md
run `lfv snap` first, or discard changes manually.
```

**实际执行**：满足前置后，若当前 tree HEAD 没有任何标签、且不是目标 Tree Snapshot 的祖先，先自动为它打上 `tree:detour/<anchor-short>/<n>` 标签（命名规则见 §4.5）（tree 面没有分支，这是让离开的 tip 保持可达的唯一手段）。然后对 `<TS-ish>` 涉及的每一个文件，都执行一遍 `lfv rewind <file> <FS-ish>`，其中 `<file>` `<FS-ish>` 从 `<TS-ish>` 中推导（清单条目经 tree 反向引用定位 file-id）。同时将 tree HEAD 指向 `<TS-ish>`。

**效果**：将整个工作树还原为目标 Tree Snapshot 所记录的状态，并推进 tree HEAD。对每个跟踪文件采用 FF 优先策略：若目标内容已可经由某个现有分支 HEAD 访问，则直接切换分支，不新建；否则创建新的分支用于保留原分支指向，将原分支定向至目标快照。文件当前路径与清单路径不同时，同时把盘上文件移回清单路径。目标快照之后新增的文件从工作区删除；快照中存在而当前已 untracked 或 deleted 的文件将字节写回磁盘，交由下次扫描接管。详细的逐文件算法见 design §4.12。

### 4.6 标签

| 命令 | 说明 |
| ---- | ---- |
| `lfv tag <file> <FS-ish> <name>` | 给某文件快照打标签。 |
| `lfv tags <file>` | 列出该文件的所有标签。 |
| `lfv tag-delete <file> <name>` | 删除文件标签。 |
| `lfv tag --tree <TS-ish> <name>` | 给 Tree Snapshot 打标签（存储为 `tree:<name>`）。 |
| `lfv tags --tree` | 列出所有 tree 标签。 |
| `lfv tag-delete --tree <name>` | 删除 tree 标签。 |

**分支名与标签名的命名规则**（两者共用一套规则，在创建时校验，不符合直接拒绝创建）：

- 非空；不以 `/` 开头或结尾；
- 不含控制字符，不含 `:`（命名空间隔离）；
- 不含 YAML 指示符字符 `#?,[]{}&*!|>'"%@` 与反引号本身，也不含 `\`；
- 开头与结尾不含空白字符；
- 整体不等于 `-`，也不匹配 `/^-\s/`（连字符后紧跟空白）；
- 不是命名空间保留字 `file`、`snap`、`tree`、`work`、`blake3`；
- 同一文件内分支名与标签名不得重名：`lfv tag` 创建时检查该文件的分支表，`lfv branch-rename` 创建时检查该文件的标签表；树标签在树面全局唯一。

以上规则同样适用于 LFV 隐式创建的保留分支名与 `tree:detour/...` 标签。各条约束的来源与序列化后果见 design §2.1.1。

标签一旦创建不可改指向；删除后同名可重建。

### 4.7 合并与变基

合并与变基操作均**仅针对文件面（file plane）分支**，与树面（tree plane）无关。两者都以**同的祖先** 为基点，通过 4-way merge 逐步重放变化，并引入冲突处理机制。即内容层 file-object 是对齐的坐标。4-way 指的是定位基点时涉及两条分支上两条不同的基点快照（base-snap-ours 与 base-snap-theirs）；每一步的内容合并本身仍是三路（base-object、ours、theirs）。

> [!note] 注意
> `lfv merge` 或 `lfv rebase`，与 `lfv relink` 有区别：
> - `lfv merge` 或 `lfv rebase` 是同一个文件的历史版本操作，进行4路合并。
> - `lfv relink` 涉及2个文件 ID 合并为一个文件 ID，不涉及3路合并内容，它只简单地把一个文件的当前分支历史挂到另一个文件的当前分支历史末尾。

**参数约束**：`<target>` 只接受 `<FS-ish>`（分支名或快照 id）——file-object 不携带历史信息，无法作为历史操作的参数（见 §3.2）。分支名等价于该分支当前 HEAD 所在的快照。

`lfv merge` 与 `lfv rebase` 均需先定位同的祖先（§3.3）。若无法找到任何同的祖先，报错拒绝操作：

```
error: no common ancestor found between branch 'main' and 'feature'
       cannot merge/rebase without a shared content base.
```

若该文件已存在一个进行中的 merge/rebase/pick（即 `REPLAY.yaml` 已存在），`lfv merge`、`lfv rebase`、`lfv merge --pick` 均拒绝再次发起，提示先 `--continue` 或 `--abort`。

#### 4.7.1 `lfv rebase <file> <FS-ish>`

在本分支的 base-snap-ours 节点处插入目标分支的 base-snap-theirs 到 HEAD 之间的历史。

**语义**：
- 先将本分支 HEAD rewind 到同的祖先节点对应的快照 base-snap-ours（分支名保持不变，旧 HEAD 位置由新建的 `rebase/<anchor-short>/<n>` 分支来保留，命名规则见 §4.5）
- 再以目标分支的 base-snap-theirs 为基点，把目标分支从 base-snap-theirs 之后到其 HEAD 的每一步 file-object 变化，依次以 4-way merge 在本分支上重建，产生一系列新的 file-object 和对应 Snapshot
- 最后把本分支原来从 base-snap-ours 之后到原 HEAD 的每一步 file-object 变化，依次以 4-way merge 继续应用在本分支上，产生一系列新的 file-object 和对应 Snapshot，本分支 HEAD 推进到新链末端

**执行步骤**：

1. 定位同的祖先（§3.3）；分别收集目标分支从 base-snap-theirs 之后到其 HEAD、以及本分支从 base-snap-ours 之后到原 HEAD 的有序 Snapshot 列表。
2. 将本分支 HEAD rewind 到 base-snap-ours（保留当前分支名，新建分支指向原 HEAD）。
3. 对目标分支的待应用序列中的每一步 `snap_i`（按时间顺序）：
   - base = `snap_{i-1}` 的 file-object（首步用 base-object）
   - ours = 本分支当前落脚点的 file-object
   - theirs = `snap_i` 的 file-object
   - 执行 4-way merge，产生新 file-object，追加新 Snapshot
4. 对本分支原来从 base-snap-ours 之后的待重放序列，重复步骤 3 的处理逻辑，将本分支的历史重建在目标分支历史之后。
   - 若遇**冲突**：暂停，输出冲突标记，等待用户解决后继续（见 §4.7.3）
   - 若遇**环回**：提示用户二选一——rewind 跳过该步（接受后自动继续），或 abort 回滚（见 §4.7.4）

**历史保留**：rebase 产生的是新的 Snapshot 链，不修改 Snapshot 层的任何已有记录；原 HEAD 位置由新建分支保留，随时可查。

#### 4.7.2 `lfv merge <file> <FS-ish>`

在本分支的当前 HEAD 处插入目标分支的 base-snap-theirs 到 HEAD 之间的历史。

**语义**：
- 保持本分支当前 HEAD 不动（分支名与位置均不变）
- 以同的祖先 base-object 为基准，把目标分支从 base-snap-theirs 之后到其 HEAD 的每一步 file-object 变化，依次以 4-way merge 应用在本分支 HEAD 之上，产生一系列新的 file-object 和对应 Snapshot，本分支 HEAD 推进到新链末端

**执行步骤**：

1. 定位同的祖先（§3.3）；收集目标分支从 base-snap-theirs 之后到其 HEAD 的有序 Snapshot 列表（待应用序列）。
2. 对待应用序列中的每一步 `snap_i`（按时间顺序）：
   - base = `snap_{i-1}` 的 file-object（首步用 base-object）
   - ours = 本分支当前落脚点的 file-object
   - theirs = `snap_i` 的 file-object
   - 执行 4-way merge，产生新 file-object，追加新 Snapshot
   - 若遇**冲突**或**环回**：处理方式同 §4.7.3 / §4.7.4

#### 4.7.3 冲突处理

4-way merge 产生冲突时，LFV 暂停操作，将冲突标记写入工作区文件：

```diff
 <<<<<<< ours (main)
 本分支的内容
 =======
 目标分支的内容
 >>>>>>> theirs (feature / snap:01HXYZ)
```

注：本示例每行加了前导空格，是为了防止版本管理把这里当成冲突。实际使用时没有这个前导空格。

二进制文件无法写冲突标记：LFV 同样暂停，用户以 `--continue --ours` 或 `--continue --theirs` 选择一侧内容继续，或 `--abort`。

用户手动编辑解决冲突后，执行：

```bash
lfv merge --continue   # 或 lfv rebase --continue
```

LFV 将解决后的工作区内容作为新 file-object，追加 Snapshot，继续下一步重放。

若用户放弃，执行：

```bash
lfv merge --abort   # 或 lfv rebase --abort
```

LFV 将分支 HEAD 恢复到操作前的原始状态，工作区内容一并还原，所有已追加的中间 Snapshot 通过分支指针回退隐藏（Snapshot 本身保留在 append-only 日志中，但不再被任何分支引用）。

`--continue` / `--abort` 建议带上 `<file>`（如 `lfv merge --continue docs/note.md`）。省略时 LFV 查找进行中的操作：恰好一个文件处于 merge / rebase 中则作用于它；多个文件同时处于进行中时报错并要求指定 `<file>`；没有进行中的操作也报错。

#### 4.7.4 环回处理

重放过程中若某步产生的 file-object 已在当前分支历史中出现（违反分支对象唯一性，design §4.13），LFV 暂停并提示：

```
warning: step snap:01HXYZ produces content already present in branch 'main'
         (object blake3:abc123...)
options:
  [r] lfv rewind to skip this step and continue rebase/merge
  [a] abort — restore branch to original state
```

- 选择 **rewind**：LFV 对当前步执行 rewind（将中间历史移入 `detour/<anchor-short>/<n>` 分支），然后**自动继续**后续重放步骤，无需用户再次确认。
- 选择 **abort**：同 §4.7.3 的 abort 语义，分支 HEAD 恢复原始状态。

#### 4.7.5 `lfv merge --pick <file> <snap-id>`

把单个快照的变化作为一步重放到当前分支 HEAD 之上（相当于 git cherry-pick）：base = 该快照父快照的 object（父为 `null` 或父 object 为 `null` 时 base 为空内容），theirs = 该快照的 object，ours = 当前 HEAD 的 object；执行一次 4-way merge，产生新 file-object 并追加 Snapshot。冲突与环回处理同 §4.7.3 / §4.7.4。`<snap-id>` 必须属于同一文件。冲突或环回后同样用 `lfv merge --continue` / `lfv merge --abort` 续接或回滚（§4.7.3/§4.7.4），不需要额外的 `--pick` 标记；省略 `<file>` 时的规则见 §4.7.3。§5.9 的内容删除流程依赖此命令。

### 4.8 维护

| 命令 | 说明 |
| ---- | ---- |
| `lfv gc` | 回收未被任何快照引用的对象。 |
| `lfv gc --purge` | 回收没有被引用的快照和未被任何快照引用的对象，并提示这一个一高危操作，无法找回数据。可达性定义见 design §3.11。 |
| `lfv verify` | 校验对象存储的完整性（重算 hash 比对）；逐行重算快照 `digest`；遍历每个文件的所有分支，检查分支对象唯一性（design §4.13），违反则报告数据完整性错误；检查 Tree Object 引用的对象是否存在。 |
| `lfv rebuild-index` | 丢弃并从真理源（objects/、snapshots.log、各 yaml）重建 `index.db`（design §3.10）。不触碰 HEAD、分支、标签与 config。 |
| `lfv export <file> [--format zip\|tar] -o <out>` | 导出某文件的全部历史为独立归档，便于迁移。归档内容为该 file-id 目录及其可达的 File Object，目录结构与 `.lfv` 一致。 |
| `lfv import <archive>` | 导入 `lfv export` 生成的归档：把其中的 file-id 目录（`meta.yaml`、`HEAD`、分支/标签表、`snapshots.log`）与可达的 File Object 合并进当前仓库，file-id 保持不变，对象按内容 hash 去重合并。导入前逐条校验快照 `digest`，损坏的归档报错拒绝，不部分导入。若该 file-id 在当前仓库已存在则报错（正常情况下不会发生，ULID 全局唯一）。**导入后不修改工作树、不登记状态表行**——文件历史进入仓库但处于"未激活"状态，需要用 `lfv revive <file-id>` 按需恢复到工作树。内部执行步骤见 design §4.16。 |

## 5. 典型工作流规范

LFV 采用"**默认全跟踪**"策略：工作目录下的文件，正常情况下都应当处于跟踪状态。为此，相关命令执行前会对工作树做**惰性扫描**，自动响应 OS 层的新建和删除事件，无需后台守护进程。

### 5.1 第一次使用

```bash
cd /path/to/A
lfv init
# 可选：创建 .lfvignore 静态禁止跟踪的路径
echo "node_modules/" >> .lfvignore
echo "*.tmp" >> .lfvignore

lfv status
# -> 惰性扫描（design §4.3）：可跟踪的新文件被自动 track（打上跟踪标志，无 Snapshot）
#    所有文件均显示为 `A` 标志，在状态表中为 modified 状态

lfv snap -m "initial snapshot"   # 对所有 modified 文件批量拍照，产生首条 Snapshot
```

### 5.2 日常编辑

```bash
# 编辑 docs/note.md ...
lfv status docs/note.md         # 查看是否有未保存变更
lfv snap docs/note.md -m "add chapter 2 outline"
lfv log docs/note.md
```

### 5.3 对比

```bash
lfv diff docs/note.md
lfv diff docs/note.md snap:01HXYZ
```

### 5.4 回溯到旧版本（自动分支）

```bash
lfv log docs/note.md
lfv rewind docs/note.md snap:01HXY0
# -> 先自动新建分支 rewind/7RQ2M9KA/1 指向 main 原来的 HEAD（历史完整保留）
# -> 再把 main 的 HEAD 移到 snap:01HXY0，工作区内容随之恢复
# 之后的 snap 继续落在 main 上；旧路线随时可 `lfv switch docs/note.md rewind/7RQ2M9KA/1` 回去
```

### 5.5 改名 / 移动

```bash
# 场景 A：通过 LFV 改名（推荐，原子）
lfv mv docs/note.md docs/notes/2026-05/note.md
# -> 追加一条 rename 快照，path 由旧变新，object 不变

# 场景 B：先用 OS 改了名，未改内容 —— 自动识别即可
mv docs/note.md docs/notes/2026-05/note.md
lfv status
#   R   file:01HA7BCD   docs/note.md -> docs/notes/2026-05/note.md
#                    auto-detected (identical content hash)
lfv snap                        # 一并落盘所有已识别变更

# 场景 C：先用 OS 改了名，又改了内容 —— 自动识别失败，手动续接历史
mv docs/note.md docs/notes/2026-05/note-v2.md
$EDITOR docs/notes/2026-05/note-v2.md
lfv status
#   D   file:01HA7BCD   docs/note.md
#                    file missing on disk; possibly moved
#   A   file:01HA7EEE   docs/notes/2026-05/note-v2.md  [newly tracked]
lfv relink file:01HA7EEE --onto file:01HA7BCD
# -> file:01HA7EEE 的内容（当前路径 + object）作为 file:01HA7BCD 历史的下一条 Snapshot 追加
# -> 事件类型由 file:01HA7BCD 末尾快照与新 Snapshot 对比推导（通常为 R+M）
# -> file:01HA7EEE 退役（状态表行取消，snapshots.log 保留）
```

### 5.6 删除与复活

```bash
lfv delete docs/old-note.md     # 删除工作树文件，并将状态表标记为 modified
lfv snap -m "remove old note"   # 追加 object = null 的 Snapshot（也可留给下次无参 lfv snap 批量处理）
lfv log docs/old-note.md        # 历史依然可查
lfv revive docs/old-note.md     # main 的 HEAD 回到删除前的快照并恢复内容到工作树；
                                # 删除事件由自动新建的 revive/<anchor-short>/1 分支保留
```

### 5.7 同名新文件接续旧历史

```bash
# 场景：docs/old-note.md 曾被删除（该 file-id 的最新 Snapshot 为 object = null），
#        现在在同一路径新建了一个文件

lfv status
#   A   file:01HA7EEE...   docs/old-note.md  [newly tracked]
#                       note: this path previously existed as file:01HA7BCD (deleted)
#                       to continue its history instead, run:
#                         lfv relink file:01HA7EEE --onto file:01HA7BCD

# 选择 A：新文件就是新文件，与旧历史无关，直接 snap
lfv snap docs/old-note.md -m "new document"

# 选择 B：新文件是旧文件的延续，接续旧历史
lfv relink file:01HA7EEE --onto file:01HA7BCD
# -> file:01HA7EEE 的内容作为 file:01HA7BCD 历史的下一条 Snapshot 追加（revive 事件）
# -> file:01HA7EEE 退役（状态表行取消，snapshots.log 保留，不留活跃痕迹）
```

### 5.8 全局快照（tree plane）

```bash
# 编辑完当前版本的所有文件后，先确保全部 snap
lfv snap -m "finish chapter 3"

# 创建 tree-snapshot，标记里程碑
lfv snap --tree -m "第三章完成"
lfv snap --tree --tag v1.0 -m "第一版完成"   # 同时打标签

# 查看 tree 历史
lfv log --tree
# snap:01HABC  2026-05-30 10:00  "第一版完成" [tree:v1.0]
# snap:01HXYZ  2026-05-17 09:21  "第三章完成"

# 回到某个 tree-snapshot（工作区内容整体还原）
lfv rewind tree:v1.0
lfv rewind snap:01HXYZ
```

### 5.9 内容删除

当历史中包含不当内容（如涉及隐私、安全问题或不再需要的敏感版本）时，LFV 提供一套**三步工作流**来实现彻底清除。本节阐述原因与树快照在此流程中的交互方式。

> [!warning] `lfv gc --purge` 是不可逆操作。执行后所有被清理的对象和快照均无法恢复，仅在明确需要时运行。

#### 5.9.1 三步工作流

对每条包含不当内容的分支，按顺序执行：

1. **重建历史**：`rewind` 到不当内容所在快照的父快照，开一条新分支；在新分支上逐个 `merge --pick` 原分支的后续快照，必要时编辑去除不当内容，形成新的 append-only 快照链。整个过程完全在现有原语范围内，不违反 append-only。
2. **删除原分支**：将含有不当内容的原分支删除。原分支上的快照链失去所有可达入口，进入悬空状态。
3. **物理清除**：执行 `lfv gc --purge`，删除所有悬空快照及其关联的 File Object 和 Tree Object。

对于不当内容出现在多条分支的情况，对每条相关分支分别执行步骤 1–2，最后统一执行一次 `gc --purge`。

#### 5.9.2 树快照的处理

Tree Snapshot 同样受到 append-only 约束，不能直接修改。但当 Tree Object 指向的 File Object 被清除后：

- Tree Snapshot 本身**保留**在 append-only 日志中（仍可作为时间线节点），但其引用的 File Object 已物理消失，`lfv show` / `lfv diff` 查询时会显示 `[object missing]`。
- Tree Object 若不再被任何 Tree Snapshot 引用（即其对应的工作目录快照也通过 rewind + gc --purge 清理），则作为悬空对象一并删除。
- **推荐做法**：在执行内容删除前，先用 `lfv rewind tree:<tag>` 将 tree HEAD 移到不含不当内容的里程碑上，避免树面留下断裂的引用。若不慎产生断裂引用，`lfv verify` 会报告 Tree Object 指向不存在的 File Object。

悬空的快照和 File/Tree Object 默认保留（用户可能后续找回），仅 `gc --purge` 执行真正的物理删除。

### 5.10 跨设备同步与协作

直接通过 NAS / 网盘把整个工作目录（含 `.lfv`）拷贝/同步到另一设备即可。LFV 本身 **不解决并发写入冲突**——同步工具的责任是确保不会同时修改 `.lfv`。

与其它位置的仓库进行内容协同开发，可以使用远程仓库。具体内容略，规范将来再明确。

## 6. 技术选型设计

- **语言**：Rust 2024 edition。
- **CLI 框架**：`clap` v4，使用 derive 风格定义命令树。
- **错误处理**：`thiserror`（库级别定义错误类型） + `anyhow`（CLI 顶层收尾）。
- **哈希**：`blake3`（速度快，足够强）。
- **标识符**：`ulid`（生成 file-id、snap-id 用的 ULID）。
- **压缩**：`zstd` level 3；默认 `min_bytes` 4 KiB、`max_bytes` 16 MiB、`reject_if_larger: true`（详见 design §3.2）。
- **元数据序列化**：`serde` + 严格 YAML（人类可读的配置/元数据，格式约束见 §3.5） + `serde_json`（snapshots.log 行格式）。`.lfv` 下所有配置与元数据文件统一使用 `.yaml` 后缀。
- **索引存储**：`rusqlite`（嵌入式 SQLite，单文件 `index.db`）。
- **文本 diff 与三路合并**：`diffy`（Myers diff、unified 格式、`merge` 三路合并带冲突标记）。
- **时间**：`jiff`。
- **日志**：`tracing` + `tracing-subscriber`，CLI 通过 `-v/-vv` 控制级别。
- **测试**：`assert_cmd` + `predicates` + `tempfile` 做集成测试；单元测试就近放在 mod 中。

> 选型原则：优先使用已被 Rust 生态广泛验证的库，避免引入不维护的 crate。所有依赖在 1.0 之前每季度审视一次。

## 7. 路线图（粗略）

- **v0.1** ：`init` / `config` / `track` / `untrack` / `snap` / `status` / `log` / `show` / `list` / `rebuild-index`；以及 `branches` / `switch` / `rewind`——`lfv snap` 的环回提示要求用户执行 `rewind`，三者必须与 `snap` 同期落地。
- **v0.2** ：`diff` / `mv` / `delete` / `revive` / `relink` / `branch-rename` / `branch-delete`。
- **v0.3** ：`tag` 系列、`export` / `import`、`gc`、`verify`。
- **v0.4** ：`merge` / `rebase` / `merge --pick`、冲突处理。
- **v0.5** ：tree plane（`snap --tree` / `log --tree` / `rewind <TS-ish>` / `tag --tree`）、性能优化。
- **v1.0** ：稳定 CLI 语义，文档完整，跨平台 CI 通过。
- **v1.1** ：远程仓库，完成协同任务。
