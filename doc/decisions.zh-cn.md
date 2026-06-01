# LFV — 关键设计决策（Key Design Decisions）

本文档记录 LFV **为什么** 这样设计。每条说明一个具体设计选择背后的理由，包括接受的权衡和否定的替代方案。

## 1. 文件为中心

所有快照都是对单个文件进行归档，全局快照命令本质上只是对多个文件逐一执行单文件快照操作。

LFV 的目的就是将文件作为独立单元来跟踪。只允许全局快照而不允许独立文件快照，相当于要求跨文件耦合——而正是 `GIT` 与 `LFV` 的区别。

## 2. 回溯不破坏历史

快照是 append-only 且始终可达，任何"回到过去"的操作都通过**新建分支**实现，绝不会让 HEAD 之前的快照变得不可达。

破坏性撤销是用户错误和数据丢失的持续来源。通过让每次回溯都变为分支创建，确保历史不被破坏。代价（分支管理略复杂）与安全保证相比微不足道。

`snapshots.log` 永不重写，天然支持备份（增量拷贝始终有效）、审计（验证器可以从头重放）和断电恢复（尾部的不完整写入可被检测并截断，不污染早期条目）。

原地修改的替代方案毫无收益，却破坏以上三个属性。

## 3. 内容寻址 + 去重

在存储机制上与 GIT 保持一致。内容寻址的键值，便是 `file-id` ，它是文件的持久身份，是唯一稳定引用。

`file-id` 在 track 时分配，贯穿文件整个生命周期（含删除后），是文件消失、路径变更后唯一仍然有效的引用手段。文件删除或改名后，路径不再可解析。用户和脚本需要一个稳定的句柄来查询历史（`lfv log`）、恢复内容（`lfv revive`）或续接历史（`lfv relink`）。

路径是常规场景下的便利别名，所有接受 `<file>` 的命令同时接受路径或 `f_*`。两者在所有命令中均被接受，用户在日常操作中无需被迫查询 `file-id`。

## 4. 跟踪策略

工作目录下的文件，正常情况下都应当处于被跟踪状态。LFV 在相关命令前做增量惰性扫描：新建文件自动 `track`，OS 删除的跟踪文件被标记为 `modified` 并显示为 `D`，下次 `lfv snap` 会按当前 FS 状态追加 Snapshot。

LFV 的心智模型是"我有一个目录，其中每个文件都有版本历史"，默认全跟踪决策主要是这个心智更符合大多数场景。逐文件显式 opt-in 反转了这个模型：用户必须在每次新建文件后记得运行 `lfv track`，忘记就丢失历史。默认全跟踪符合预期行为，与备份工具的工作方式一致。需要排除的文件通过 `.lfvignore`（静态排除列表）或 `lfv untrack`（动态逐文件 opt-out）处理。

不进入跟踪通常由 `.lfvignore` 与 `config.yaml` 决定，但二者的意义不一样：

- `.lfvignore` 匹配的路径对 LFV 完全不存在——不进状态表，不参与扫描，不出现在任何输出中。这与 `.gitignore` 的语义一致，用户心智负担低。
- `config.yaml` 的 untracked 记录用户运行时的动态管理，它在增加删除跟踪时，会有明确的输出提示。
 
而数据库中状态表的跟踪状态，由上2个真理源决定。状态表通常仅反映"当前盘上存在"的文件（除 `D` 例外），避免表内出现大量"幽灵行"，使 `lfv status` 输出始终反映真实的工作树状态。

之所以做两种不跟踪区分，主要是运行中输出信息的要求不一样。

## 5. Snapshot

Snapshot 为版本记录，大体对应 GIT 的 commmit。

Snapshot id 采用 ULID 以保证可读性与时间排序；防篡改职责交由独立的 `digest` 字段承担，由 `lfv verify` 校验（详见 `design.md §5.2`）。

内容寻址 id（如 Git 的 SHA 哈希）将身份与防篡改耦合在一起，被迫在"不透明哈希（体验差）"和"可读字符串（无法作为完整性证明）"之间取舍。LFV 将这两个关注点分离：ULID id 便于粘贴、时间有序、内容无关；`digest` 字段提供独立的防篡改检测。`lfv verify` 可以在不触碰 id 的情况下校验每条 Snapshot。

在快照中 ULID 与路径完全解耦；rename/move 被记为**不可变事件**。将路径作为每条 Snapshot 的字段，Snapshot 链就成为文件位置与内容的唯一完整真理源。杜绝可变 `aliases` 列表导致的真理源分裂。

路径每次改名要么打断历史连续性，没有 ULID，则需要一张可变的别名表，它使得真理源分裂。

## 6. 删除只是普通 Snapshot 的一种取值，历史永不丢失

`lfv delete` 把文件从磁盘删除，把状态表的文件标记为 `modified`。`lfv snap` 时检查目标文件的 FS 状态，文件不存在就追加一条 `object = null` 的 Snapshot，并从状态表删除该文件记录。所有历史 Snapshot 完整保留，随时可 `lfv revive`。

LFV 的删除是**状态转移**，不是擦除。擦除历史违反 append-only 保证（决策 3），使 `lfv revive` 无从实现。将删除视为 `object=null` 的 Snapshot 在结构上与模型其他部分完全一致——存储层无需特殊处理。

两步设计（标记 `modified` → 在 `lfv snap` 时确认）给用户一个修正窗口：文件仍处于 `D` 状态时，可运行 `lfv mv` 将其重新归类为改名，避免误删。

## 7. Tree Snapshot 与 File Snapshot 对称

`lfv snap --tree` 创建 **Tree Snapshot** 的方式与 `lfv snap` 创建 File Snapshot 完全一致：向 append-only 日志追加一条不可变事件记录，以 ULID 寻址，有单一 parent 指针，有用于防篡改的 `digest` 字段，并指向一个 Tree Object。

Tree Object 是内容寻址的 YAML 清单——按路径排序的 `{path: file-object-hash}` 条目列表，包含当前 HEAD 下所有有效（非 null）object 的跟踪文件。Tree Object 与 File Object 存放于同一 `objects/` 目录，按内容去重。两次内容完全相同的工作目录状态产生相同的 Tree Object hash，`objects/` 里只有一份物理条目。

Tree Snapshot 存放于 `.lfv/trees/snapshots.log`，拥有独立的标签集合，通过现有命令的 `--tree` 变体管理（`lfv log --tree` 等）。仓库内可以不存在任何 Tree Snapshot；tree 面完全可选。

`lfv snap --tree` 要求所有 modified 跟踪文件必须已经完成快照（先执行 `lfv snap`，或加 `--snap-all` 参数）。这确保 Tree Object 是从每个文件的当前 HEAD 构建的，不会静默地产生过期的 tree 快照。

**为什么不在 `lfv snap --tree` 内部自动 snap 文件**：将文件快照与 tree 快照分开，保持用户意图的明确性。Tree Snapshot 是刻意标记的里程碑；把可能很多个文件的快照作为副作用自动触发，会用无意的快照条目遮蔽各文件的独立历史。

## 8. 单分支 object hash 唯一性不变量与环路防止

**不变量**：在某个文件历史的任意单条分支上，同一个 file-object-hash 最多出现一次。

若无此约束，某条分支可能回访之前出现过的内容状态，在内容层图中产生反向边（环路）。环路使内容层的渲染产生歧义：同一个节点在时间线上出现两次，其注释（message、时间戳）变得不明确。

**机制**：`lfv snap` 计算新内容 hash 时，若发现它与当前分支某个祖先的 object hash 相同，则拒绝追加快照，改为输出提示：

```
warning: content of docs/note.md matches ancestor snap_2 on branch B
suggestion: run `lfv rewind docs/note.md snap_2` to re-anchor
            this will preserve the intermediate history as branch detour_B
```

用户执行 `lfv rewind` 后：
1. 当前分支重命名为 `detour_<原分支名>`——中间历史完整保留。
2. 在原分支名上新建一条快照，其 parent 指向**匹配祖先的父快照**（而非祖先本身），使新快照与祖先在 Snapshot 链上是两个独立节点，尽管它们的 object hash 相同。

**为什么指向祖先的父，而非祖先本身**：若新快照的 parent 是该祖先，则两者 object hash 相同、parent 也相同——在 Snapshot 链中无法区分，渲染时无法表达两者之间的有效边。

**为什么是提示而非自动 rewind**：中间历史（detour）可能是用户有意为之，值得在归档前审阅。将决策暴露给用户，与 LFV 一贯"不静默地重构历史"的哲学一致。

**`lfv verify` 强制校验**：单分支唯一性不变量是可检验的属性。`lfv verify` 遍历每条分支，将任何违反报告为数据完整性错误，而非警告。同一分支上出现相同 object hash 的第二次，意味着环路防止流程被绕过，正常操作下不应发生。

## 9. 拓扑视图设计

每条 Snapshot 只有一个 parent 指针，整体形成有向森林——无论是 File Snapshot 链还是 Tree Snapshot 链，均如此。分支分叉后独立演化，不存在拓扑意义上的合并点（即物理上不存在多 parent 的 Merge Snapshot）。这与 GIT 的 DAG 不同。

Snapshot 有向树拓扑与“内容合并（Content Merge）”是解耦的，坚持快照的有向树拓扑，并不等于排斥单文件在内容层面的合并与对齐。

所以，在用户视角，视图是快照与对象都向用户暴露：用户以 Object 为显示上的节点，从 Snapshot 链接看这些节点的继承关系，在用户视图上是 DAG 拓扑。

- **Snapshot 层**（存储层）：永远是有向树。每条 Snapshot 恰好有一个 parent 指针。这一层是物理存储的内容，结构永不改变。
- **内容层**（派生层）：节点为 object hash，边由 Snapshot parent 关系投影到 object 身份而来。若无额外约束，此层可能出现环路（某分支回访了之前出现过的 object hash）。决策 8 通过强制"单分支 object hash 唯一性"不变量，在此层消除环路，使其成为 DAG。

LFV 物理上排斥 Git 式的 DAG 拓扑（多父节点），纯粹是为了保持底层的**极致简单与历史追溯的绝对确定性**：

- **存储与索引极简**：`snapshots.log` 仅需 optional `parent` 字段，快照历史天然是单向链表的物理分叉。
- **算法极简**：拓扑排序退化为简单的 O(N) 线性回溯，彻底规避了 Git 中复杂的图拓扑排序（Topology Sort）和环路检测算法。
- **历史绝对纯净与可读**：有向树保证了任何快照的`祖先路径`是唯一确定的。
- **无双线并行歧义**：Git merge 后分支在拓扑上存在两条路径，无法区分哪条是真正的主线。LFV 的单父拓扑从根源上杜绝此问题，这样可以保证各个分支的颗粒度可以是不同的。

### 优缺点对比（有向树拓扑 vs. DAG 拓扑）

| 维度 | Git 的 DAG 拓扑 | LFV 的有向树（Directed Tree）拓扑 |
| :--- | :--- | :--- |
| **拓扑定义** | 每个节点（Commit）可有 1 到多个父节点（多父节点用于记录 merge） | 每个节点（Snapshot）最多只能有 1 个父节点（即 parent 指针唯一） |
| **底层复杂度** | 极高。需处理复杂的交叉路径和多路径可达问题。 | 极低。数据结构为单链表分叉树（有向森林），存储、追溯、GC 均极轻量。 |
| **合流机制** | **拓扑级合并 (Merge Commit)**：通过多父指针在物理图上建立合流点。 | **重构/内容级合流 (Rebase/Content Merge)**：物理上不设多父合流点，靠 parent 指针重定向或单亲快照追加内容。 |
| **历史可读性** | 极易被污染。单文件历史线（`git log <file>`）会被大量无关提交和交叉线模糊。 | 绝对纯净。祖先路径唯一确定，完美还原该单文件随时间演进的线性脉络。 |
| **场景适用性** | 多人协作、多文件高频合并的分布式大型软件工程项目。 | 单人、单文件、离线、本地，如笔记、配置、NAS 备份等单文件历史追踪。 |

### 复合视图的决策原因

LFV 认为“内容合并”与“历史拓扑合并”是两件不同的事情。用户合并文件时，真正关心的是内容是否已经统一，而不是多个历史分支是否在快照拓扑上收敛为同一个节点。在LFV的界面下，只要 Object 一致就认为合并完成，不需要在快照上对齐。而 git 要在 commit 合一上做文章。这种分离，极大地方便了合并算法和用户的流程理解。

## 10. Tree 面的设计原则

### 10.1 tree-id 使用内容 hash，不使用 ULID

tree-id（Tree Object 的身份标识）采用 `blake3(规范化清单字节串)` 内容 hash，而不是像 file-id 那样分配独立的 ULID。

一个仓库逻辑上只有"一棵 tree"——它是工作目录当前所有跟踪文件的集合，随内容变化而变化，不需要一个稳定的"这棵 tree 是谁"的身份。内容 hash 作为 tree-id 有两个好处：自然去重（两次内容完全相同的工作目录状态共享同一 Tree Object），以及省去为 tree 维护独立身份（ULID + meta.yaml）的开销。这与 file-id 的设计不同——file 需要跨改名/移动保持稳定身份，因此必须是与路径解耦的 ULID；tree 不需要跨内容变化保持身份，因此内容 hash 就足够了。

### 10.2 Tree 没有分支，只有标签和 HEAD

Tree Snapshot 链没有分支集合，只有：标签（`t:` 前缀，全局唯一）、单条 HEAD 指针（当前所在的 Tree Snapshot）。

分支的核心价值在于支持"同一文件的多条独立演化线"——这是 file 面的典型需求（用户需要在不同分支上实验不同内容）。Tree 是里程碑式的全局快照，其使用模式是线性推进，不需要也不应该有多条并行演化线。引入 tree 分支只会增加心智负担，而不带来实质好处。

**tree HEAD 存储在单独文件 `.lfv/trees/HEAD` 中**：tree HEAD 是不可从 Snapshot 链重建的当前状态（它记录"用户当前位于 tree 历史的哪个节点"，而非哪条 Snapshot 是最新的）。`index.db` 里的其他状态（分支指针、file HEAD）在 `rebuild-index` 时可以从 Snapshot 链派生重建；tree HEAD 一旦丢失无法重建，因此应独立于可重建的 `index.db`，以单独文件持久化，避免在 `rebuild-index` 时被意外覆盖。

`lfv switch <branch>` 只针对 file 分支，不提供 tree 的切换操作（因为 tree 没有分支）。tree 的位置变更只通过 `lfv rewind t:<tag|snap-id>` 操作。

### 10.3 tree rewind 不直接操作 config、分支或状态表

`lfv rewind t:<tag|snap-id>` 执行时，只做两件事：更新 `.lfv/trees/HEAD`，以及对工作区文件执行字节级操作（写入或删除）。它不直接调用 `track`/`untrack`/`revive` 命令，不修改 `config.yaml` 动态 untracked 列表，不修改任何分支指针或状态表。

`config.yaml`、分支和状态表属于 `track`/`untrack`/`delete`/`revive` 命令的职责边界。tree rewind 越过这个边界直接操作，会使用户难以预测哪些命令会对 config 产生副作用，破坏命令职责的清晰性。

字节操作之后，现有的惰性扫描机制（§6.8）会在下次 `lfv status` 或 `lfv snap` 时自然感知变化并驱动状态更新。tree rewind 产生的"新增文件"和"消失文件"与 OS 直接操作文件产生的效果完全等同，用户的心智模型无需特殊化。

### 10.4 tree rewind 对每个 file 采用 FF 优先策略

`lfv rewind t:<tag|snap-id>` 对每个需要还原的 file，不直接执行 rewind（会产生新分支），而是先检查是否存在 Fast-Forward 路径：

- **FF 路径**：若某条分支的当前 HEAD 的 object hash 等于目标 hash，直接 `switch` 到该分支，不新建分支。优先选当前分支（无切换成本）；若当前分支不匹配，选最近创建的匹配分支。
- **rewind 路径**：否则执行标准 rewind，自动新建分支（`rewind/<snap-short>/<n>`）。

tree rewind 是批量操作，可能同时影响数十个文件。若每个文件都无条件新建分支，产生的分支噪音会极大干扰用户对各文件历史的阅读。FF 路径在实践中覆盖大多数 tree rewind 场景（因为 tree-snapshot 通常紧随各文件 snap 之后创建，此时各文件 HEAD 的 object 就是 tree-object 里记录的 object），使零分支污染成为常态。只有真正需要回溯到非 HEAD 位置时才新建分支，与决策 2（回溯不破坏历史）保持一致。

这与 git 的 Fast-Forward merge 机制同构：在能 FF 的情况下不产生额外节点，只在必要时才分叉。

### 10.5 Tree Object 使用 YAML 块序列而非 JSON

Tree Object 清单采用 YAML 块序列格式（`- "path": hash`），而非 JSON 数组（`[{"path":...,"object":...}]`）。

**被否决的方案**：JSON 数组。JSON 的规范化需要额外约定（无多余空白、键顺序固定、无尾随逗号），实现时必须使用受控的序列化器而非普通 `to_string()`，规范化要求与格式规范分离，容易因实现疏漏产生不一致的 hash。

**选择 YAML 块序列的理由**：

**存储格式即规范化格式**：每行 `- "path": hash` 完全确定（引号、排序、单一换行符），blake3 对字节串直接摘要，无需额外规范化步骤。规范化约束体现在写入规则中，而非附加的序列化协议上。

**路径作为键名的安全性**：YAML 中裸键（unquoted key）对 `#`、`[`、`{`、`,`、`&`、`*` 等字符有特殊含义，路径字符集与之存在冲突。双引号键名彻底消除所有特殊字符问题——包括含空格的路径（在 Windows/NAS 场景极为普遍）。路径始终以 Unix 分隔符 `/` 存储，不含 `\`，键内唯一需转义的字符是 `"`，实际路径中几乎不出现，因此引号带来的复杂度可忽略不计。

**外部兼容性**：该格式是合法的 YAML 子集，外部程序可直接用标准 YAML 解析器读取。LFV 内部亦可用单行正则 `/^- "(.+)": (\S+)$/` 逐行解析，不依赖完整解析器。JSON 同样有外部兼容性，但 YAML 在字符节省和解析简便性上略优。

**字符效率**：每条目约节省 20% 字符（YAML 行约 35 字符 vs. JSON 对象约 45 字符）。Tree Object 体积通常低于 `min_bytes` 压缩下限，以 `.raw` 存储，字符效率直接对应存储效率。

## 11. 存储对象仅两类

仓库内真正不可变的存储对象分为 **Object** 与 **Snapshot** 两大类，每类各有两种子类型。其余概念均为可变索引（详见 `design.zh-cn.md §4.2`）。

| 大类 | 子类型 | 内容 | 寻址方式 |
| --- | --- | --- | --- |
| Object | **File Object** | 文件原始字节（压缩后） | `blake3(原始字节)` |
| Object | **Tree Object** | 按路径排序的 `{path: file-object-hash}` 清单（YAML） | `blake3(规范化清单字节串)` |
| Snapshot | **File Snapshot** | 单个文件的事件记录 | `snap_<ULID>` |
| Snapshot | **Tree Snapshot** | 工作目录整体状态的事件记录 | `snap_<ULID>` |

两种 Object 子类型均存放于同一个 `objects/` 目录，按内容 hash 去重，完全不可变。两种 Snapshot 子类型均为 append-only，格式相同（JSON Lines），但分别存放于不同目录：
- File Snapshot: `.lfv/files/<file-id>/snapshots.log`
- Tree Snapshot: `.lfv/trees/snapshots.log`

**为什么现在需要 Tree Object**：LFV 的应用场景包括对整个工作目录进行里程碑式快照（例如"这本书的某个完整版本"）。Tree Object 使这成为一等操作，同时保持核心不变量——内容身份由 hash 决定，而非快照 id。两次内容完全相同的工作目录状态产生相同的 Tree Object hash，`objects/` 里只有一份，与两个文件内容相同时共享同一 File Object 的机制完全一致。

**为什么 Tree Object 存储 file-object-hash 而非 file-snapshot-id**：LFV 历史图的节点身份是内容（object hash），不是元数据记录（snapshot id）。Tree Object 引用 file-object-hash，从端到端都是内容寻址；若引用 snapshot id，则树的身份会与偶发的元数据（message、时间戳）耦合，破坏去重和内容节点语义。

## 12. 对象压缩双阈值

`min_bytes`（floor，默认 4 KiB）与 `max_bytes`（ceiling，默认 16 MiB）分别处理过小与过大的文件；`blake3` 始终对原始字节计算；跳过或无效压缩的对象以 `.raw` 存储，压缩对象为 `.zstd`（详见 `design.md §5.1`）。

单一阈值无法同时处理两个边界情况。过小的文件压缩后可能因帧开销反而变大；过大的文件在压缩时会产生内存峰值（需要将整个文件读入内存）。双阈值明确区分这两种情况，`reject_if_larger` 则兜底处理已压缩的二进制（如图片）等无压缩收益的情形。`blake3` 始终对原始字节计算，确保 hash 与存储格式无关。

## 13. 以 SQLite 作为索引

状态表、branches、tags、文件的全局快速索引等，需要随机更新的元数据放入 `index.db`。

它只是缓存作用，真理源数据走文件。

状态表和分支指针是**可变的**，需要高效的随机访问读取和原子更新。纯文件方案（如每文件一个 YAML）在小规模下可行，但在数千个被跟踪文件时性能急剧下降。SQLite 提供 ACID 事务、高效索引查找和单文件部署模型——无守护进程，无网络。历史 Snapshot 链则是 append-only 且顺序访问，用 JSON Lines 平文件更简单、足够用。

## 14. `lfv mv` 与 `lfv relink`

`lfv mv <src> <dst>` 只做路径操作，`<dst>` 不允许为 `file-id`。`lfv relink <f_src> --onto <f_dst>` 专门处理历史续接。

若允许 `lfv mv <src> <f_dst-id>` 将路径操作与历史续接混用，用户会自然地以为操作完成后"留下来的 `file-id` 是 `<dst>` 那个"——而实际上续接后存活的是 `--onto` 一侧（即 `f_dst`），`f_src` 被退役。这一心智混淆几乎必然导致误操作。

`lfv relink` 还解决了 `lfv mv` 根本无法处理的第二个问题：**合并另一个文件的历史**。典型场景是 `<f_src>` 的内容是 `<f_dst>` 的超集，用户不需要同时保留两个 `file-id`，希望将 `f_src` 的完整快照链续接到 `f_dst` 上，然后退役 `f_src`。这是语义上的历史整合操作，与单纯的路径重命名是两个不同层次的概念，强行合并只会使两个命令都变得难以理解。

## 15. 文件的树索引使用 `tree_file_refs` DB 表

每个 file 在哪些 Tree Snapshot 中出现的反向引用存入 `index.db` 的 `tree_file_refs` 表，而非每个 file-id 目录下的独立文件。字段：`tree_snap_id`（`snap_<ULID>`）、`file_id`、`file_object`（blake3 hash）。真理源为 `.lfv/trees/snapshots.log` + `.lfv/objects/`，`rebuild-index` 时重建。

**被否决的替代方案**：每个 file-id 目录下维护一个 YAML 文件（`.lfv/trees/.yaml`），key = `snap_<ULID>`，value = file-object-hash。此方案在文件数量多时会产生大量小文件写入，且没有事务保证（写入中崩溃会产生不一致）。

**采用 db 表的理由**：`index.db` 提供 ACID 事务，单表查询效率高，且 `tree_file_refs` 本就是纯缓存性质（真理源在 `.lfv/trees/snapshots.log` + `objects/`），放入 db 与其他可重建索引语义一致，`rebuild-index` 时统一重建，不会出现某个 file 的反向引用漏写的情况。

**snap_id 作为主键（而非标签名）的理由**：每个 Tree Snapshot 都有 message，即使没有标签也有意义。snap_id 作为主键保证所有 Tree Snapshot 都被反向引用，渲染时再去 `.lfv/trees/snapshots.log` 动态查询标签：有标签则显示标签名，无标签则显示 message，二者都有意义。"打不打标签"只影响展示，不影响历史的完整性。

## 16. 命名空间隔离：用户输入不允许包含 `:`

tree 标签以 `t:` 为前缀（内部管理），file 有隐式 `f:` 命名空间（通常不显示）。**用户在输入分支名、标签名时不允许包含 `:`**，由此实现命名空间隔离，避免用户输入与内部前缀冲突。

**为什么**：如果允许用户输入 `t:foo`，CLI 无法区分这是用户有意引用 tree 标签，还是用户真的想创建一个名为 `t:foo` 的 file 标签。通过禁止 `:` 出现在用户输入中，命名空间的所有权边界清晰：带前缀的引用总是内部生成的，不带前缀的引用总是用户输入的。

## 17. 小工具优先

CLI 子命令清晰、可组合、可脚本化；不为图形化或服务化做过早抽象。

LFV 面向自动化、shell 脚本和与其他工具的集成（同步脚本、编辑器、CI）。优先保持干净的 CLI 约定使工具可组合。GUI 或守护层可以构建在稳定的 CLI 之上；反过来则很痛苦。。

## 未来可能的扩展

- 简单HTML5界面，可视化某文件的分支树。
- 钩子（hook）系统，例如保存前自动 snap。
- 跨仓库的对象池共享。
- 与 git LFS / NAS 厂商 API 的桥接。

## 待决问题（Open Questions）

下列问题会在开发过程中根据实际反馈决定：

1. **改名自动识别阈值**（`rename.autodetect`）：仅在内容哈希完全一致时识别，还是允许"相似度 ≥ N%"的近似匹配？后者复杂度高，目前只做精确匹配。
2. **`lfv revive` 默认恢复点**：从最新的 `object != null` 快照恢复，还是要求用户显式指定 `<snap>`？倾向前者作为默认，并允许用户覆盖。
3. **同名新文件续接历史**：`lfv revive` 与同名新文件续接历史时，是否需要额外的安全确认或 `--force`，以避免用户把语义无关的新文件误接到旧历史上。
