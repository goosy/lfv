# LFV Tree 层设计草稿（待并入正式文档）

> 本文件记录对原 design.zh-cn.md 中 tree 层设计的调整，经讨论确认后将并入
> `design.zh-cn.md` 和 `decisions.zh-cn.md`。

---

## 核心定位

在 LFV 中，file 是一等公民，tree 是二等公民：

- file 的分支和标签归属于某个 file-id 的命名域，各个 file 可以复用同名分支/标签。
- `lfv status` 只检查 file 状态，不考虑 tree 是否与工作区"匹配"。
- file-object 和 tree-object 是历史视图的**呈现节点**，而非快照本身。
- 每个节点若对应多个快照，显示当前分支的快照注释（详细面板可补全其他分支信息）。

---

## tree 层概念

因为 LFV 的应用场景也包括对整个工作目录进行里程碑式快照（例如"这本书的某个完整版本"），
所以需要一个全局快照机制。**全局快照**是面向用户的术语，底层实现称为 **tree-snapshot**，
它指向一个 **tree-object**（存储对象之一）。

---

## tree-object

- tree-object 本质上是工作目录所有被跟踪文件的文件对象集合（`{path, file-object-hash}` 清单）。
- 清单只包含当前 HEAD 下有有效 file-object 的跟踪文件（即排除状态为 deleted 的 file）。
- **tree-id = tree-object 的 blake3 hash**，内容寻址，与 file-object 一样存放于 `objects/`。
- 全局只有一个 tree（不需要追踪"同一个 tree"的稳定身份），因此不再需要 ULID 形式的 tree-id。
- 一个仓库里可以不存在任何 tree-object 和 tree-snapshot，tree 层完全可选。
- tree-object 的存储格式为 YAML 格式的 `{path: file-object-hash}` 数组，按 path 字典排序。

---

## tree-snapshot

- 命令：`lfv snap --tree -m "信息"` 创建一个 tree-snapshot。
- `lfv snap --tree` 要求所有被跟踪文件均已完成快照（工作区无 modified 文件），否则报错。
  此限制确保 tree-object 是从每个文件当前 HEAD 构建的，不会产生过期的 tree 快照。
- tree-snapshot 有单父指针（`parent`），形成可视历史链（有向树）。
- tree-snapshot **没有分支**，只有标签和 HEAD 概念。
  - 标签全局唯一，不允许复用，显示和存储时加 `t:` 前缀。
  - `lfv switch branch-name` 只针对 file 分支，不操作 tree。
- tree-snapshot 的历史视图渲染逻辑与 file 历史相同：以内容节点（tree-object hash）为准，
  而非以快照 id 为准。
- tree-snapshot 的 id 格式与 file-snapshot 相同：`snap_<ULID>`，不加额外前缀（全局唯一，无需区分）。

### 前置校验

`lfv snap --tree` 执行前检查所有 modified 跟踪文件。若存在 modified 文件则拒绝执行，
提示用户先执行 `lfv snap`（或加 `--snap-all` 同时批量处理，待 v0.x 决定是否支持）。

---

## tree-head

- tree-head 记录当前 tree-snapshot 链上的位置，即"用户当前处于 tree 历史的哪个节点"。
- **不可由 index.db 重建**，存放于单独文件 `.lfv/trees/HEAD`（存 snap id 或为空）。
- 当前工作目录内容不一定与 tree-head 所指快照一致（类似 git 的 detached HEAD）。

---

## 仓库结构（tree 相关部分）

```
.lfv/
└── trees/
    ├── snapshots.log   # append-only，JSON Lines，tree-snapshot 记录
    ├── tags.yaml       # tree 标签表，key = "t:<name>"，value = snap_<ULID>
    └── HEAD            # 当前 tree-head，存 snap_<ULID> 或为空
```

tree-object 与 file-object 共用 `.lfv/objects/` 存储，按内容 hash 去重。

---

## tree rewind

### 前置校验

`lfv rewind t:<tag|snap-id>` 执行前，检查工作区是否存在 **modified 但尚无 file-object 的文件**
（即新建、修改或删除但还未 `lfv snap` 的文件）。若存在，**拒绝执行**，提示用户先处理：

```
error: the following files have unsaved changes (no file-object yet):
  M  docs/draft.md
  A  docs/new.md
run `lfv snap` first, or discard changes manually.
```

原因：这类文件没有 file-object，tree rewind 无法恢复它们，强行继续会造成工作内容永久丢失。

### 执行流程

1. 将 `.lfv/trees/HEAD` 更新为目标 tree-snapshot 的 snap id。
2. 读取目标 tree-snapshot 指向的 tree-object，得到清单（`{path, file-object-hash}`）。
3. 按以下三类分别处理所有 file：

**分类 A：在 tree-object 清单中的 file（需还原内容）**

对每个 file，以 `target_hash` 为目标，执行 **FF 优先的 rewind**：

- **FF 路径**：若某条分支的当前 HEAD 的 object hash == `target_hash`
  → 优先选当前分支（无切换成本）；当前分支不匹配则选最近创建的匹配分支；
  → 直接 `switch` 到该分支，**不新建分支**。
  提示：`[FF] docs/note.md → branch main`
- **rewind 路径**：否则
  → 执行标准 rewind，**自动新建分支**（命名规则 `rewind/<snap-short>/<n>`）。
  提示：`[rewind] docs/note.md → new branch rewind/01HXYZ/1`

**分类 B：不在 tree-object 清单中、但仓库中已有 file-object 的 file**
（即 tree-snapshot 创建后新增并已快照过的 file）

→ **仅删除工作区文件**，不动 `config.yaml` 动态 untracked 列表，不动任何分支或状态表。
  后续扫描（§6.5 自动 delete 机制）发现该路径消失，自动将其标记为 `modified (D)`。
  用户执行 `lfv status` 时会看到 `D` 状态，下次 `lfv snap` 时按正常 delete 流程追加
  `object = null` 的 Snapshot。历史与 file-object 完整保留，随时可 `lfv revive`。
  提示：`[deleted] docs/new-chapter.md  (not in target tree-snapshot)`

**分类 C：在 tree-object 清单中、但当前已 untracked 或 deleted 的历史 file**
（即该 file 在 tree-snapshot 时是有效的，后来被 untrack 或 delete）

→ **仅将文件字节写回工作区**，不修改 untracked 列表，不新建分支，不修改任何状态表。
  后续扫描机制接管：
  - 若原 file-id 是 **deleted** 状态：扫描在同一路径发现新文件，走 §6.2
    "同名文件友好提示"路径，分配新 file-id，`lfv status` 显示 `M`/`A` 并附带提示
    "此路径曾存在旧 file-id，可用 `lfv relink` 续接历史"。
  - 若原 file-id 是 **untracked** 状态：扫描发现路径在 config 动态 untracked 列表中，
    `lfv status` 显示 `~`（untracked），用户可自行决定是否 `lfv track`。
  提示：`[restored] docs/old-chapter.md  (was absent, file content written back)`

4. 所有 file 处理完毕后，输出汇总。

**设计要点**：

- tree-snapshot 只记录 file-object-hash，不记录各 file 的分支信息。分支是 file 的局部概念，tree 层不感知。
- tree rewind **不直接操作 `config.yaml`、分支、状态表**，这些都属于 `track`/`untrack`/`delete`/`revive` 命令的职责。tree rewind 只做两件事：操作工作区文件字节，以及对分类 A 的 file 执行 FF/rewind。其余状态变化全部交给现有扫描机制自然产生。
- 分类 B 的副作用（后续 `lfv snap` 会追加删除快照）是 tree rewind 的预期语义：用户执行 tree rewind 就应有这个心理准备，与 OS 直接删除文件后的处理方式完全等同。
- FF 路径是零分支污染的理想路径。实践中大多数 tree rewind 会走 FF，因为 tree-snapshot 通常在各文件 snap 之后立即创建。

---

## file 反向引用：`tree_file_refs` 表（`index.db`）

`index.db` 中的 `tree_file_refs` 表记录每个 file 在哪些 tree-snapshot 中出现，
供 `lfv log <file>` 在历史视图上显示全局快照信息。

### 表结构

```sql
CREATE TABLE tree_file_refs (
    tree_snap_id  TEXT NOT NULL,  -- snap_<ULID>
    file_id       TEXT NOT NULL,  -- file-id
    file_object   TEXT NOT NULL,  -- blake3:...
    PRIMARY KEY (tree_snap_id, file_id)
);
```

### 真理源与重建

真理源为 `.lfv/trees/snapshots.log` + `objects/` 中的 Tree Object 内容。`tree_file_refs` 是纯缓存，
`rebuild-index` 时通过遍历所有 Tree Snapshot、展开各 Tree Object 清单重建。

### 写入时机

每次 `lfv snap --tree` 创建 tree-snapshot 时，LFV 展开 Tree Object 清单，向 `tree_file_refs` 逐 file 插入一行。

### 渲染逻辑

`lfv log <file>` 渲染历史节点时，通过 `tree_file_refs` 查找该文件对象 hash 对应的
tree-snapshot，再去 `.lfv/trees/snapshots.log` 查询标签和 message：

- 若有 `t:` 标签，显示标签名；
- 无标签则显示 tree-snapshot 的 message；
- 无关联则不显示。

示例：
```
snap_01HXYZ  M  docs/note.md   "add chapter 2"
             └─ tree: "第三章完成" [t:v1.0]
snap_01HWWW  M  docs/note.md   "fix typo"
             └─ tree: "日常归档 2026-05-30"
```

---

## `lfv log --tree` 输出格式

CLI 视图显示 tree-snapshot 链，每个节点展示 message、标签、时间戳：

```
snap_01HABC  2026-05-30 10:00  "第三章完成" [t:v1.0]
snap_01HXYZ  2026-05-17 09:21  "日常归档 2026-05-17"
snap_01HWWW  2026-05-01 14:00  "初稿完成" [t:draft-1]
```

GUI 程序中再考虑展示每个 tree-snapshot 涵盖的 file 变更摘要等详细内容。

---

## `lfv tag` 对 tree 标签的支持

tree 标签命令形式：`lfv tag --tree <snap> <name>`

```bash
lfv tag --tree snap_01HABC v1.0      # 给 tree-snapshot 打标签，存储为 "t:v1.0"
lfv tags --tree                       # 列出所有 tree 标签
lfv tag-delete --tree v1.0            # 删除 tree 标签
```

创建树快照时可直接附加 `--tag` 参数：

```bash
lfv snap --tree --tag v2.0 -m "第二版完成"
```

`t:<name>` 作为命名空间前缀的命令形式留待后续讨论，暂不入文档。

---

## 命名空间隔离

- `t:` 前缀为 tree 标签的命名空间，由 LFV 内部管理。
- `f:` 为 file 的隐式命名空间，默认不显示。
- **用户不能输入任何包含 `:` 的标记符**（分支名、标签名均不允许），由此实现命名空间隔离。
- 设计之初不考虑其他命名空间（如 `remote:`），预留扩展空间。

---

## 与原设计的主要差异（对照 design.zh-cn.md）

| 项目 | 原设计 | 新设计 |
| ---- | ------ | ------ |
| tree-id | ULID，稳定身份 | blake3 hash，内容寻址 |
| tree 存储位置 | `.lfv/trees/<tree-id>/` | `.lfv/trees/`（固定单目录） |
| tree 分支 | 有，独立分支集合 | **无分支**，只有标签和 HEAD |
| tree HEAD 存储 | `index.db` | `.lfv/trees/HEAD`（单独文件，不可重建） |
| `.lfv/trees.yaml` key | `t:<tag>`（tag 做键） | `snap_<ULID>`（snap id 做键） |
| `lfv switch` | 可操作 tree 分支 | 只针对 file 分支 |
| tree rewind 分类 B | 写入 config untracked 列表 | 仅删除工作区文件，交扫描机制处理 |
| tree rewind 分类 C | 自动执行 lfv revive | 仅写回文件字节，交扫描机制处理 |
| tree rewind 整体 | 对每个 file 直接 rewind | 前置校验 + FF 优先 + 三类分类处理 |
| 反向引用存储 | `files/<file-id>/trees.yaml`（YAML 文件） | `index.db` 的 `tree_file_refs` 表（db 缓存） |
