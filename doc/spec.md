# LFV — Lightweight File Versioning

> This document is the English specification (spec) of the LFV project: it defines what to build, why, and with which behaviors and constraints. For the Chinese version, see `spec.zh-cn.md`.
> The implementation design (how to build it: storage structure, internal mechanisms, engineering structure) is in `design.md`; this document references its sections as `design §x`.

---

## 1. Background and Motivation

`git` is an excellent version-control tool, but its design target is **repository-scoped** version management — best suited for a logically cohesive collection of source files: a single commit can span multiple files, and branches record the holistic evolution of an entire working tree.

Yet daily work also contains another archetypal scenario, with the following characteristics:

- Each file is **logically independent**, with no coupling to any other;
- Each file has **its own line of history**, unrelated to other files;
- Users care about the evolution, comparison, and rollback of **one specific file** along a timeline.
- The semantic need to **"commit multiple files at once"** is not required;

Typical examples:

- **NAS file sync systems**: each backed-up file has its own history, but cross-file atomic commits are unnecessary.
- **One-note-per-file repositories**: each note evolves independently; changes to note A should not "pollute" note B's history.
- **Design drafts / contracts / single-document archives**: each document tracked individually.
- **Configuration file directories**: each config file evolves independently, without affecting others.

Existing approaches to this scenario are generally unsatisfying:

- **Simple backup**: copies old versions only, with no way to see the purpose of each version;
- **Retain the last N versions**: lacks semantic information; no diff, no annotation;
- **Whole-directory git repository**: mixes the evolution of unrelated files, buries individual file history, and violates the "file-centric" mental model;
- **Cloud drive version history**: typically unusable offline, and lacks branching, comparison, or portability.

## 2. Project Goals

The goals of LFV (Lightweight File Versioning):

1. Provide an `lfv` CLI so users can, as intuitively as with `git`, operate on all files under a directory (i.e. the repository):
   - Selectively track a file;
   - Create annotated snapshots of that file;
   - View the list of historical snapshots of that file;
   - Diff any two snapshots (or a snapshot against the current content);
   - Rewind to a historical snapshot — on rewind, **a new branch is created automatically to preserve the old line**, never destroying existing history.
2. Each file has its **own independent**: branch history, branch set, and tag set.
3. Each branch of a file must be a single-directional **non-branchable** path, without mid-path divergence or merge.
4. Provide good portability: the `.lfv` directory can be copied or synced together with the working directory and behaves consistently after migration.
5. Compact data storage: content-hash deduplication and compression avoid the storage bloat of naive backups.
6. Distributed collaboration: coordinated sync with local or network repositories, providing basic push / pull / remote / merge semantics.

In order to stay "lightweight", the following are **out of scope**:

- **Distributed communication protocol**: LFV does not constrain the underlying protocol of network repositories; it only specifies the interface for obtaining a `.lfv` directory through a remote reference. Implementors may extend drivers to support address formats such as `ssh:uname@host:~`, `https://other.dns/somerepo.lfv`, `https://webdav.host/somerepo`, etc.
- **Merge scope limitation**: `merge` / `rebase` operate only on the branches of a single file; there is no cross-file coordination.
- **Staging operations**: always "the working tree is the part a snapshot applies to"; when versions must be separated, rewind and modify snapshots instead.
- **Replacing git**: source-code engineering scenarios should continue to use git; LFV serves only the "file-centric" scenario.
- **Graphical interface**: CLI only; a GUI should be a separate project.

## 3. Terminology Spec

### 3.1 Basic Terminology

| Term | Description |
| ---- | ---- |
| Repository | The `.lfv` directory under the working directory; carries the metadata and object storage of all tracked files. |
| Working Tree | The working directory itself (excluding `.lfv`); where the files the user actually operates on live. |
| Tracked File | A file brought under repository management by `lfv track`. Internally identified by a `file-id` (ULID) as its stable identity, fully decoupled from the path. |
| File Status | For a path already within LFV's view, the status is `unmodified`, `modified`, or `untracked`. (see design §4.3) |
| Object | A content-addressed storage unit for file content or for a global reference; deduplicated by content hash, so different files may share the same object. |
| Snapshot | A photograph of one file, or of all files globally, at a moment in time: content pointer + path + metadata (message, timestamp, author, parent snapshot). |
| Branch | A snapshot chain under one tracked file; the default branch is `main`. Each file's branch namespace is independent of the others. |
| HEAD | The current branch name and latest snapshot pointer of a tracked file. |
| Tag | An optional readable name for a snapshot, used to reference a version stably. |
| Action | An operation that changes file state. Explicit actions: `track`, `snap`, `untrack`; implicit actions: `modify` (the user edits a file), `auto-track` / `auto-delete` (LFV responds to OS events automatically during a scan, see design §4.3). |
| LFV-visible | The files remaining in the working directory after directory pruning by `.lfvignore` matches. |

### 3.2 Storage Objects: File Object, Tree Object, File Snapshot, Tree Snapshot

LFV's storage objects consist of two **fully immutable, content-addressed** types — File Object and Tree Object — both stored in `.lfv/objects/` and deduplicated by content hash. What they have in common: written once, never modified. Integrity can be verified by recomputing the hash with `lfv verify`.

LFV's snapshots are an **append-only event record layer**, addressed by ULID, recording "what happened at a certain moment". They also come in two types: File Snapshot and Tree Snapshot. Their structures are highly symmetric, and they serve **two independent views**.

All snapshots are append-only; existing snapshots are never rewritten.

> [!note] Immutability principle for snapshots and objects
> LFV's append-only guarantee is that snapshots "**within the traversable range**" cannot be modified. Unreachable (dangling) snapshots are exempt from this constraint and can be safely physically deleted. File Objects and Tree Objects, as content-addressed units, have even stronger invariance — once written, never changed. This principle is the safety foundation for deletion operations such as `lfv gc --purge`; see §5.9.

> [!note] Relationship between File Objects and history
> A File Object is only a content unit and carries no historical information — it has no parent-child relationship, does not know which branch it belongs to, and does not know at which point in time it was produced. Historical information is carried entirely by the Snapshot chain.
> Any operation that needs historical context (`lfv log`, `lfv merge`, `lfv rebase`) must take `<FS-ish>` (a snapshot id, or a branch name / tag name with file context) as its argument, never a bare file-object hash — the latter identifies only "what the content is", not "where in history it lives".
> All forms of `<FO-ish>` (see the preamble of §4) may be used wherever only the content is needed.

#### 3.2.1 File Object

A File Object is the content-addressed storage unit for a single file's raw bytes. Its identity is the `blake3` hash of the file's raw content — files with different paths or histories share the same File Object if their content is identical. A File Object is fully immutable: written once, never modified. Storage details (compression, bucketing, deduplication) are described in design §3.2.

`file-id` is the **stable identity** of a tracked file inside the repository:

- A separate ULID is assigned **at first track**.
- It has **no derivation relationship whatsoever** with the file path and does not change on rename or move; a `file-id` remains valid after the file is deleted.
- A file's "current path" is maintained by the mutable index (§3.4) and continuously derived and updated from events on the Snapshot chain — the mutable state in `index.db` can all be rebuilt from Objects, the Snapshot chain, and the metadata files under `.lfv`; `HEAD`, the branch table, the tag table, and `config.yaml` record user intent, cannot be derived from the snapshot chain, and are each a source of truth in their own right.

#### 3.2.2 Tree Object

For users, this is the repository snapshot; the term "repository snapshot" is used in the user interface, while this specification unifies it as Tree Object.

This object stores the content manifest of all valid tracked files in the working directory at a given moment: a list of `{path, file-object-hash}` entries in lexicographic path order, forming a canonical manifest. The tree-id is the `blake3()` of that manifest byte string, i.e. content-addressed. Two identical working-directory states produce the same Tree Object hash, and `objects/` stores only one copy. When there are no valid tracked files in the working directory the manifest is empty — this is still a valid Tree Object.

Tree Objects share the same `objects/` directory with File Objects, and their identity computation, deduplication, and storage mechanics are identical (see design §3.2). Tree Objects are usually very small and are normally stored as-is.

#### 3.2.3 File Snapshot

Each File Snapshot is a complete event record for a tracked file at a given moment. Core fields:

- `id`: `snap:<ULID>`, time-ordered, globally unique, never rewritten.
- `parent`: parent snapshot id; `null` for the first snapshot on a branch.
- `path`: the file's relative path in the working tree at the time of this snapshot (the path is a field of the snapshot, not the identity of the file).
- `object`: the blake3 hash of the referenced File Object; `null` when the file does not exist (a delete event).
- `digest`: the blake3 hash of the canonical serialization of every field of this record except `digest`, used by `lfv verify` for tamper detection.

Additional informational fields — message, author, time, and so on — are described in design §3.3. Snapshots do not record tags; a tag is an external mutable pointer (design §3.8).

File Snapshots have the following characteristics:

- **The event type is not stored explicitly.**
- **The owning branch is not recorded**; a snapshot records only a historical event.
- The entire lifecycle of a file is its File Snapshot chain.
- "Rename/move", "add", "content change", and "delete" are meaningful only in the user interface, and can be distinguished simply by comparing fields with the parent snapshot. They are derived on the read side from the `(parent.path, parent.object, path, object)` quadruple (add / modify / rename / rename+modify / delete / revive); see design §3.3.1 for details.
- Each File Snapshot has only one parent pointer, so the whole forms a **directed tree (forest)**: once a branch diverges it evolves independently, and there is no topological merge point.

#### 3.2.4 Tree Snapshot

Each Tree Snapshot is a milestone record of the whole working directory, storing mainly:

- `id`: `snap:<ULID>`, append-only, never rewritten
- `object`: tree-object pointer, pointing to a Tree Object (never null)
- `parent`: parent snapshot id
- `digest`: the blake3 hash of the canonical serialization of every field of this record except `digest`, used by `lfv verify` for tamper detection.

Other fields such as message, author, and time are described in design §3.4.

A Tree Snapshot is structurally symmetric to a File Snapshot, with these field differences:

- The `path` field is empty (a tree needs no path to locate).
- `object` may not be null (a tree snapshot always points to a Tree Object).

The tree plane has no concept of branches, and therefore no `branches.yaml`.

Global snapshots are optional for the user: a repository may contain no Tree Snapshot at all.

A Tree Snapshot likewise has only one parent pointer, forming a directed tree, and all Tree Snapshots are stored in a single append-only log (storage path and format in design §3.4).

Starting from a Tree Snapshot, the file-object-hash in the Tree Object manifest locates the content of a file at that moment; searching that file's Snapshot chain for entries with a matching object hash then yields the corresponding File Snapshot (the single-branch object uniqueness invariant guarantees that matching snapshots form at most one contiguous run on a branch, of which the last entry with the same path is taken). The manifest itself contains no file-id; "which file" is answered by the tree reverse-reference cache (design §3.6).

#### 3.2.5 Topology Layers

LFV history has two independent topology layers:

- **Snapshot topology layer** (physical storage): always a directed tree; each Snapshot has exactly one parent pointer, and the structure never changes.
- **Content topology layer** (derived view): a DAG whose nodes are object hashes and whose edges come from projecting the Snapshot parent relationship onto object identity (consecutive snapshots pointing at the same object collapse into a single node). Single-branch object uniqueness guarantees that this layer has no cycle on any individual branch.

That is, objects and snapshots are decoupled: the former carries only content, the latter describes topology, and one object may be traversed by multiple snapshot timelines.

### 3.3 Snapshot Co-reference Determination and Co-referent Ancestors

In LFV, different snapshots may point to the same File Object. LFV calls two such snapshots **co-referent**, and many LFV operations are built on co-reference.

Rebase and merge operations, for instance, need to locate the co-referent ancestor: walking up the Snapshot chains of the two branches to find the nearest node, or pair of nodes, that appeared in both histories and are co-referent:

- **base-snap-ours**: the Snapshot in this branch's history that points to the shared file-object
- **base-snap-theirs**: the Snapshot in the target branch's history that points to the same file-object
- **base-object**: the file-object both point to (identical content, stored once)

The co-referent ancestor snaps of two branches may be the same snapshot, or different ones (when the two branches independently went through identical content), so base-snap-ours and base-snap-theirs are frequently not the same snapshot. A merge operation based on such a co-referent ancestor — especially in the more general case where the co-referent ancestors are not the same snapshot — is what LFV calls a **4-way merge**.

### 3.4 Mutable Index

LFV maintains a rebuildable **Mutable Index** as a cache of working-area state, so that no command has to scan the whole filesystem; the recommended implementation is an embedded database. Storage objects (Objects), the Snapshot chain, and the metadata files under `.lfv` (`HEAD`, the branch table, the tag table, `config.yaml`) together form the repository's source of truth, so a corrupted index can be rebuilt from it at any time (`lfv rebuild-index`, §4.8). See design §3.6 for details.

### 3.5 YAML Constraints on Metadata Files

This section constrains the **mutable, user-intent metadata files** under `.lfv/` — `config.yaml`, `meta.yaml`, `branches.yaml`, `tags.yaml`, `trees/tags.yaml`, `REPLAY.yaml` (see design §3.7, §3.8). `HEAD` and `trees/HEAD` are plain text and are not subject to this section. Tree Object is also encoded as YAML, but it is a content-addressed object whose canonical form is defined separately in design §3.5, aimed at byte-exact matching; the rules here do not apply to it.

Goal: what LFV writes must be legal YAML that any standard YAML parser can read correctly, while the writing style is tight enough to be read and written by a minimal parser that does not depend on a general-purpose YAML library. Whether LFV's own implementation ends up using a full YAML crate or a hand-written minimal one, this section is a hard constraint on the on-disk format, not an optional parser behavior.

**Semantic subset**: only mappings, sequences, strings, numbers, booleans, and null; no anchors, aliases, complex tags, or implicit type inference.

**Writing conventions**:

1. Block style only; no flow style (`{...}`/`[...]`) anywhere.
2. Indentation is fixed at 2 spaces per level; sequences always start with `- `.
3. For a fixed-schema structure (`meta.yaml`, `config.yaml`, the `Step` in `REPLAY.yaml`), every field is always emitted in the order its struct declares them; an empty `Option` is written explicitly as `~`, never omitted.
4. Comments are allowed only as full trailing tokens at the end of a line; because a value is either a bare safe one or mandatorily quoted (see below), "everything from the first unquoted `#` to end of line" is always a safe comment boundary.
5. No blank lines within the file; exactly one trailing newline at end of file.

**Strings fall into three categories by how controllable they are, handled differently**:

- **User-controlled strings** — content comes from the user and LFV places no restriction on the character set (`RepoPath`, `config.yaml`'s `user.name`, a snapshot's `author`). **Always double-quoted**, escaped inside the quotes by the standard YAML double-quoted-string rules (the same as `serde_json` string escaping: only `"`, `\`, and control characters are escaped; non-ASCII is written as raw UTF-8). `RepoPath` already forbids `"` and `\` (§4.2), so this rule needs no real escaping in practice for paths; `user.name`/`author` have no character restriction and do need full escaping under this rule.
- **User-proposed, LFV-approved** — branch names and tag names (`BranchName`/`TagName`). The user names them, but they only come to exist after passing LFV's creation-time validation, so their character set can be restricted, in exchange for staying unquoted. The exact character rules are in §4.6; where each constraint comes from is explained in design §2.1.1.
- **LFV-controlled** — `snap:<ULID>`, `file:<ULID>`, `blake3:<hex>`, RFC 3339 timestamps, `true`/`false`, integer version numbers, and the like. The character set is defined by LFV itself and known safe; written bare, unquoted and unescaped.

## 4. CLI Functional Spec

The parameter conventions below all ultimately resolve to one of the four storage objects (File Object, Tree Object, File Snapshot, Tree Snapshot). Branch names and tag names live in **a namespace that is independent per file** (§2, goal 2), so they may not appear on their own and must always carry file context.

- `<file>` locates a tracked file (not one of its versions), in two spellings:
  - a path: `a`, `./a` and `../a` are relative to the current working directory; `/a/b`, starting with `/`, is a repository-absolute path (rooted at the repository root, not an operating-system absolute path). Operating-system absolute paths and drive letters are not accepted, so the same spelling works on every platform. Every path is normalized to a path relative to the repository root, and one that resolves outside the repository is an error; a path that has disappeared from disk resolves to the last non-retired file-id that held that path in history (among several candidates, the one whose HEAD snapshot has the latest `created_at`, and failing that the largest snap-id); once that path is taken by a new file it resolves to the new active file-id, so a file that has disappeared is best referenced by file-id;
  - `<file-id>`, i.e. `file:<ULID>`.
- `<FS-ish>` is an argument that resolves to a File Snapshot, including:
  - `<snap-id>`: a snapshot identifier `snap:<ULID>`, globally unique and carrying its own file affiliation;
  - `<file>`: the HEAD snapshot of that file's current branch;
  - `<branch>:<file>`: the HEAD snapshot of one branch of that file;
  - `<tag>:<file>`: the snapshot that a tag of that file points to.
  The form matches git's `<rev>:<path>`. Paths, branch names and tag names never contain `:`, and branch and tag names may not be a namespace keyword (`file`, `snap`, `tree`, `work`, `blake3`), so splitting at `:` is unambiguous; file names are not restricted by the keywords.
  In commands that already take a `<file>` argument (such as `lfv rewind <file> <FS-ish>`), `<FS-ish>` may omit the `:<file>` part and be written directly as a branch name, tag name, or snap-id; the resolution order is snap-id -> branch name -> tag name (a branch and a tag of the same file may not share a name, see §4.6, so this order only serves to separate them from a snap-id).
- `<FO-ish>` is an argument that resolves to a File Object, including:
  - any `<FS-ish>`: resolves to that snapshot's `object`;
  - `work:<path>`: the current working-area content, treated as a special unsaved File Object.
- `<TO-ish>` is an argument that resolves to a Tree Object, including:
  - `<tree-id>`: the content hash of a Tree Object (`blake3:...`);
  - `tree:<tag>`: the Tree Object of the tree snapshot a tree tag points to;
  - `<snap-id>`: the Tree Object corresponding to a tree snapshot (the snap-id must belong to the tree plane).
- `<TS-ish>` is an argument that resolves to a Tree Snapshot, including:
  - `<snap-id>`: a tree snapshot identifier (it must belong to the tree plane; snap-ids are globally unique and the index tells which plane one belongs to);
  - `tree:<tag>`: the tree snapshot that a tree tag points to.

### 4.1 Repository Management

| Command | Description |
| ---- | ---- |
| `lfv init` | Create `.lfv/` in the current directory. Errors if it already exists. |
| `lfv config <key> [value]` | Read or write repository configuration (e.g. `user.name`). |

### 4.2 File Tracking

| Command | Description |
| ---- | ---- |
| `lfv track [<file>]` | Add a file to tracking. When `<file>` is given, errors if the path matches `.lfvignore`; otherwise removes it from the untracked list in `config.yaml`. If that path was previously untracked rather than deleted, the original `file-id` is reused and the status table is updated to `modified` (awaiting the first `lfv snap` when there is no snapshot history). With no file argument, automatically scans all trackable files not yet in the status table and adds them to tracking. |
| `lfv untrack <file>` | Stop tracking: errors if the file is LFV-invisible; if visible, updates the dynamic untracked list in `config.yaml`; if the file exists on disk, records `untracked` in the status table (keeping its file-id so `lfv track` can reuse it), otherwise only config is touched. History is preserved and can be resumed with `lfv track` / `lfv revive`. |
| `lfv mv <old-file> <new-file>` | Migrate the tracked file at the `<old-file>` path to the `<new-file>` path. `<old-file>` accepts a path or a file-id (a file-id always corresponds to a path); `<new-file>` accepts a path only, not a file-id. If the content after the move would violate single-branch object uniqueness (design §4.13), the whole command is refused and the file on disk has not been moved at that point. Whether the actual file move is performed in the working tree is governed by design §4.6. |
| `lfv relink <src-file-id> --onto <dst-file-id>` | Splice the history of src-file-id's current branch onto the end of dst-file-id's current branch; the on-disk file becomes identified by dst-file-id and src-file-id is retired. **Operates only on the current branch of each of the two file-ids**, touches no other branch, and performs no 4-way merge content integration. Designed for the case where a new file was created by mistake; not intended as a routine command. Preconditions: src-file-id is active and its file is present on disk (`unmodified` or `modified`); dst-file-id owns no live path (it has disappeared, was deleted, or was imported but not activated) — merging two live files is refused. See design §4.7. |
| `lfv delete <file>` | Set the file to modified in the status table and delete the file itself. When a later `lfv snap` detects that it is modified and absent from disk, it appends an `object = null` Snapshot and removes it from the untracked list in `config.yaml`, updating the cached state in the status table. Its history is preserved in full and it can be revived at any time with `lfv revive`. |
| `lfv revive <file> [<FS-ish>]` | Revive a deleted file. The default restore point is the last snapshot on the current branch with `object != null`; another snapshot can be selected with `<FS-ish>`. Implemented as a rewind (§4.5): the branch name is unchanged, HEAD moves to the restore point, and the original HEAD containing the delete event is preserved by an automatically created `revive/<anchor-short>/<n>` branch; the content is written back to the last known path in the working tree. A new file with the same name that should continue the old history does not go through revive — use `lfv relink` (§5.7). |
| `lfv list [--deleted] [--all]` | List all tracked files; each record includes the path, current branch, latest snapshot id, and summary. By default lists active files only; `--deleted` also lists files whose latest Snapshot has `object = null`; `--all` additionally lists retired file-ids (relink sources) and file-ids that are untracked but still have history. |

**Path rules**: so that history recorded on any platform can be restored on every other platform, trackable paths follow the Windows file-name rules (the characters Linux and macOS forbid are a subset):

- must be valid UTF-8; paths are recorded and compared in Unicode NFC, and a file stored on disk under another normalization form (such as the NFD common on macOS) is the same path, recognized by LFV automatically;
- must not contain control characters or any of `<` `>` `:` `"` `\` `|` `?` `*` (`/` is only the separator);
- every path component is non-empty and does not end with `.` or a space;
- no path component, with its first `.` and everything after it removed, may be (case-insensitively) a Windows reserved name: `CON`, `PRN`, `AUX`, `NUL`, `COM1`–`COM9`, `LPT1`–`LPT9`, `COM¹`–`COM³`, `LPT¹`–`LPT³`.

A path that fails these rules is not registered by auto-track, which only issues a warning (the scan continues) with a hint to add it to `.lfvignore`; an explicit `lfv track <file>` on such a path is an error. Paths are stored with `/` as the separator and compared byte by byte (on case-insensitive filesystems, a change of case alone still counts as a rename). Symbolic links are neither followed nor tracked; empty directories are not tracked.

**Path input**: when parsing path arguments the CLI also accepts `\` as a separator (it can never occur in a file name), but LFV's output and hints use `/` only; `\` is not encouraged. In MinGW / Git Bash, MSYS rewrites an argument that starts with `/` into a Windows path, so a repository-absolute path must be written as `//docs/a.md` there (LFV treats several leading `/` as one).

**Cross-platform differences**: the following are left to the user; LFV only detects and reports them, and runs normally when nothing is detected.

- **Paths differing only in case**: on a case-insensitive filesystem (such as the Windows default), if the repository records two active paths that differ only in case, or an operation would write out two such paths, LFV reports an error and tells the user to enable case sensitivity for the directories involved (Windows: `fsutil.exe file setCaseSensitiveInfo <dir> enable`); otherwise it runs normally.
- **Unicode normalization collisions**: when two on-disk file names in one directory differ only in normalization form (identical once normalized to NFC, which normally only happens on Linux), LFV reports an error and the user renames one or adds it to `.lfvignore`.
- **Path length**: the Windows default path-length limit is for the user to handle (enable long-path support or shorten the path); LFV only reports the cause when a filesystem operation fails because a path is too long.

### 4.3 Status and Snapshots

| Command | Description |
| ---- | ---- |
| `lfv status [<file>]` | **Without `<file>`, lists all `modified` tracked files**; a lazy scan per design §4.3 runs first. Default output contains tracked changes only, with a `file-id` (`file:*`) on each line. For `--include-untracked` see §4.3.2. `--refresh` forces a full refresh of the scan cache. With `<file>`, only that file is shown. |
| `lfv snap [<file>] [-m <msg>]` | Create a new snapshot for a file. **Without `<file>`, batch-snapshots every tracked file in `modified` state.** Refuses if both the working-area content and the path are identical to the HEAD snapshot. If single-branch object uniqueness would be violated (design §4.13), refuses to create the snapshot and prompts the user to run `lfv rewind`. `--snap-all` is equivalent to omitting `<file>`; it merely makes the intent “batch, one shared message” visible on the command line, and may not be combined with `--tree`. For the `--tree` parameter see §4.3.3. |
| `lfv log <FS-ish>` | List the history of the branch the snapshot is on, together with tree association information (from the tree reverse-reference cache). When `<FS-ish>` is `<file>`, shows its current branch; when it is `<branch>:<file>`, shows that branch; when it is a snap-id, shows the branch containing that snapshot — preferring the current branch, otherwise the first branch containing it in branch-name order. `--all` shows the history of all branches of that file; `--graph` renders the branch topology as ASCII art; `--limit N` limits the number of entries. |
| `lfv log --tree` | List the history view of the tree plane: walk the Tree Snapshot chain, showing message, tags, and timestamp for each node. |
| `lfv show <FS-ish>` | Output the metadata of that snapshot; `--content` also outputs the object content; `--out <path>` exports the content to a file. |
| `lfv show <FO-ish>` | Without `--content`/`--out`, outputs this File Object's identity: hash, raw byte size, and every snapshot in the repository that references this hash (`snap-id` + its `path`). A File Object is a content-addressed storage unit unrelated to any single file (§3.2.1) — the same content may be referenced by snapshots of multiple file-ids and branches, so this snapshot list is not limited to the one file-id the `<FO-ish>` happened to resolve through. With `--content` / `--out`, only the object content is output, none of the above. |

Notes:
- In `lfv status`, among `modified` files, a deleted file shows as `D`, a file whose path differs from its last snapshot shows as `R`, a file with no snapshot shows as `A`, anything else shows as `M`, and a file with both content and path changes shows as `R+M`.
- During `lfv snap`, each target modified file is checked against its current filesystem state: if the file exists, an Object is written or reused and a content Snapshot is appended; if the file does not exist, a Snapshot with `object = null` is appended; `unmodified` files are skipped. A lazy scan per design §4.3 runs first; when `<file>` is given, only that file is processed (for an auto-detected rename still pending, both the old and the new path resolve to the same file).
- When a file has no snapshot yet (`A`) and has already disappeared from disk, `lfv snap` produces no snapshot and merely cancels its tracking.
- A batch `lfv snap` processes files independently: the failure of one file (a loopback, for instance) does not affect the others.

#### 4.3.1 `lfv status` Output Format (Tracked Files)

`lfv status` output looks like:

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

**Status flags**:

- `A` = add (tracked but no snapshot yet; includes the case where this path previously had a deleted file-id — see the example below)
- `M` = modified (has a snapshot; content or path differs from HEAD)
- `R` = rename (an auto-detected rename still pending; an explicit `lfv mv` appends a Snapshot immediately)
- `D` = suspected delete (a **tracked** file in `modified` state that has disappeared from disk; the next `lfv snap` appends an `object = null` Snapshot according to the current filesystem state)
- `~` = dynamic untracked (`status = untracked`, config policy + the file exists on disk)

**The `file-id` column**:

Displays `file:*` (a ULID with the `file:` prefix), abbreviated by default to the prefix plus the first 8 characters of the ULID (e.g. `file:01HA7BCD`); `--long` shows the full length. The value is assigned at track time, spans the file's entire lifecycle, and is the only stable reference token at the CLI level.

**Any command that accepts `<file>` normally accepts either a path or a `file:*` as its argument**:

- `lfv relink file:01HA7EEE --onto file:01HA7BCD` — splices the history of file:01HA7EEE (the new file-id, present on disk) onto file:01HA7BCD (the old file-id, which has disappeared); file:01HA7BCD continues as the active file-id, while file:01HA7EEE is retired but preserved.
- `lfv mv docs/old-note.md docs/notes/new.md` — a path rename.
- `lfv delete file:01HA7DEF` — a deletion can still be registered explicitly by file-id even when the file is no longer in the working tree.

#### 4.3.2 `--include-untracked` (Display Dynamic Untracked)

This is an **independent** topic from the working-tree scan (design §4.3): the flag **only changes output** and triggers no extra full-tree walk.

- **Single data source**: the `untracked` rows in the status table (all of which have been confirmed by scanning to exist on disk).
- Only LFV-visible files appear, i.e. paths matching `.lfvignore` do not (design §4.2); `.lfvignore` holds static exclusion rules, and `lfv status` never lists LFV-invisible files.
- An **Untracked** block is appended after the tracked-files block, listing entries with `~` (no `file-id`) and taking the size from `stat`; a path that has disappeared from disk has no status-table row and therefore no `~` line.

For an output example see §4.3.1; for the meaning of `~` see the status flags in that section.

#### 4.3.3 `lfv snap --tree`: Creating Tree Snapshots

```bash
lfv snap --tree -m "chapter 3 complete"
lfv snap --tree --tag v1.0 -m "first edition complete"   # tag it at creation time
lfv snap --snap-all -m "first edition complete"   # snap every modified file one by one with the same message
```

**Precondition**: no tracked file is in `modified` state; otherwise the command errors and prompts the user to run `lfv snap` first.

The `--snap-all` parameter can be used to clear `modified` beforehand: LFV runs one `lfv snap` for each `modified` file with the same `<msg>` (equivalent to the user doing it manually one by one, with each file getting its own independent snapshot in its history). The failure of one file's snap (a loopback, for instance) does not affect the snaps of the others. `modified` must be cleared before a tree snapshot is created; `--snap-all` and `--tree` cannot be used together.

**Effect**: captures the content of all tracked files in the current working directory (the file-object at each file's HEAD) as one content-addressed global snapshot, appends it to the Tree Snapshot chain, and advances the tree HEAD. If `--tag` is given, the tag is written at the same time. Internal execution steps are in design §4.11.

### 4.4 Diffing

The current working-area content can serve as a special unsaved `File Object`, referenced with the literal `work:<path>` (preamble of §4).

| Command | Description |
| ---- | ---- |
| `lfv diff <FO-ish>` | The File Object pointed to by the given FO-ish vs. the current working-area content. This amounts to omitting the second argument `work:<path>`, whose path is inferred from the current path of the file the first argument belongs to. A bare path as `<FO-ish>` resolves to the object at that file's current HEAD, so `lfv diff docs/note.md` is HEAD vs. the working area. |
| `lfv diff <FO-ish-A> <FO-ish-B>` | A comparison between two File Objects. (They may belong to different files, or be different versions of the same file.) |

Text files use a line-based diff (3 lines of context by default); binary files show metadata differences only (size, hash).

### 4.5 Rewind and Branches

| Command | Description |
| ---- | ---- |
| `lfv rewind <file> <FS-ish>` | Restore the working-area file content to the File Object of the snapshot that `<FS-ish>` points to (content only, the path is not changed; when the HEAD snapshot path differs from the on-disk path, the next `lfv snap` records an `R`). **The branch name is unchanged**: a preserving branch `rewind/<anchor-short>/<n>` pointing at the original HEAD is created automatically first, then the current branch's HEAD is moved to the target snapshot. Refuses to run while the file is in `modified` state (run `lfv snap` first, or discard the changes yourself), except in loopback mode. When used to resolve a loopback event, the user passes the matched ancestor snapshot itself, and LFV splices according to the rules in design §4.13. |
| `lfv rewind <TS-ish>` | Restore the working area to the state of the specified Tree Snapshot. Updates `.lfv/trees/HEAD`; applies an FF-priority strategy per file (details in §4.5.1). |
| `lfv branches <file>` | List all branches of that file. |
| `lfv switch <file> <branch>` | Switch that file's current branch (also updating the working-area content to that branch's head snapshot). File plane only; the tree plane is not touched. Refuses while the file is in `modified` state. When the target branch's HEAD has `object = null` (the file is deleted on that branch), the file is removed from the working tree; that path still resolves afterwards (falling back to the last non-retired file-id that held it), but once a new file takes the path it resolves to the new file-id, so referencing it stably means using the file-id. |
| `lfv branch-rename <file> <old> <new>` | Rename a branch. `<new>` must satisfy the naming rules in §4.6 and must not collide with an existing branch name or tag name of that file; renaming the current branch rewrites that file's `HEAD` as well. |
| `lfv branch-delete <file> <branch>` | Delete a branch (only the pointer is deleted; snapshots and objects are retained so they can be shared and recovered). Refuses to delete the branch the current HEAD is on; switch elsewhere with `lfv switch` first. |

**Preserved-branch naming rules**: branches created automatically by rewind-style operations are uniformly named `<kind>/<anchor-short>/<n>`. `kind ∈ {rewind, detour, rebase, revive}` states the originating operation (respectively the rewind in this section, loopback handling in design §4.13, §4.7.1, and revive in §4.2); `anchor-short` is the last 8 characters of the ULID of the preserved old HEAD snapshot; `n` increments from 1 to avoid name collisions. The tree plane has no branches; the tag `tree:detour/<anchor-short>/<n>` that `lfv rewind <TS-ish>` creates automatically for the tree HEAD being left behind follows the same naming rule, with `anchor-short` taken from the last 8 characters of the ULID of that departing tree HEAD snapshot.

#### 4.5.1 `lfv rewind <TS-ish>` Execution Flow

**Pre-check**: check whether the working area contains any tracked file in `modified` state (including `A` / `M` / `R` / `D`). If so, refuse to run:

```
error: the following files have unsaved changes:
  M  docs/draft.md
run `lfv snap` first, or discard changes manually.
```

**Actual execution**: once the precondition is met, if the current tree HEAD has no tag at all and is not an ancestor of the target Tree Snapshot, it is first tagged automatically as `tree:detour/<anchor-short>/<n>` (naming rule in §4.5) (the tree plane has no branches, so this is the only way to keep the tip being left behind reachable). Then, for every file involved in `<TS-ish>`, `lfv rewind <file> <FS-ish>` is executed, with `<file>` and `<FS-ish>` derived from `<TS-ish>` (manifest entries are mapped to file-ids through the tree reverse references). The tree HEAD is pointed at `<TS-ish>` at the same time.

**Effect**: restores the whole working tree to the state recorded by the target Tree Snapshot and advances the tree HEAD. Each tracked file uses an FF-priority strategy: if the target content is already reachable through the HEAD of some existing branch, that branch is switched to and no new branch is created; otherwise a new branch is created to preserve where the original branch pointed, and the original branch is directed to the target snapshot. When a file's current path differs from the manifest path, the on-disk file is moved back to the manifest path as well. Files added after the target snapshot are deleted from the working area; files present in the snapshot but currently untracked or deleted have their bytes written back to disk and are handed over to the next scan. The detailed per-file algorithm is in design §4.12.

### 4.6 Tags

| Command | Description |
| ---- | ---- |
| `lfv tag <file> <FS-ish> <name>` | Tag a snapshot of a file. |
| `lfv tags <file>` | List all tags of that file. |
| `lfv tag-delete <file> <name>` | Delete a file tag. |
| `lfv tag --tree <TS-ish> <name>` | Tag a Tree Snapshot (stored as `tree:<name>`). |
| `lfv tags --tree` | List all tree tags. |
| `lfv tag-delete --tree <name>` | Delete a tree tag. |

**Naming rules for branch names and tag names** (one shared rule set, validated at creation; a violation is refused outright):

- non-empty; does not start or end with `/`;
- contains no control characters and no `:` (namespace isolation);
- contains none of the YAML indicator characters `#?,[]{}&*!|>'"%@` or the backtick itself, and no `\`;
- has no leading or trailing whitespace;
- is not equal to `-` as a whole and does not match `/^-\s/` (a hyphen immediately followed by whitespace);
- is not a namespace keyword `file`, `snap`, `tree`, `work`, `blake3`;
- a branch name and a tag name of the same file may not collide: `lfv tag` checks that file's branch table at creation and `lfv branch-rename` checks its tag table; tree tags are globally unique on the tree plane.

The same rules apply to the preserved branch names and `tree:detour/...` tags that LFV creates implicitly. Where each constraint comes from, and its serialization consequences, are in design §2.1.1.

Once created, a tag's target cannot be changed; after deletion, the same name can be created again.

### 4.7 Merge and Rebase

Merge and rebase operations both target **file-plane branches only** and have nothing to do with the tree plane. Both take the **co-referent ancestor** as their base point, replay changes step by step through 4-way merge, and introduce a conflict-handling mechanism. In other words, the file-object in the content layer is the coordinate along which things are aligned. 4-way refers to locating the base point: it involves two different base snapshots on the two branches (base-snap-ours and base-snap-theirs); the content merge of each individual step is still a three-way merge (base-object, ours, theirs).

> [!note] Note
> `lfv merge` and `lfv rebase` differ from `lfv relink`:
> - `lfv merge` and `lfv rebase` operate on historical versions of the same file and perform a 4-way merge.
> - `lfv relink` merges two file IDs into one and involves no three-way content merge; it simply attaches the current-branch history of one file to the end of the current-branch history of another.

**Parameter constraints**: `<target>` accepts only `<FS-ish>` (a branch name or snapshot id) — a file-object carries no historical information and cannot be an argument to a history operation (see §3.2). A branch name is equivalent to the snapshot its HEAD currently points to.

Both `lfv merge` and `lfv rebase` must first locate the co-referent ancestor (§3.3). If no co-referent ancestor can be found, the operation is rejected with an error:

```
error: no common ancestor found between branch 'main' and 'feature'
       cannot merge/rebase without a shared content base.
```

If this file already has a merge/rebase/pick in progress (that is, `REPLAY.yaml` already exists), `lfv merge`, `lfv rebase`, and `lfv merge --pick` all refuse to start a new one, prompting the user to `--continue` or `--abort` first.

#### 4.7.1 `lfv rebase <file> <FS-ish>`

Insert the history between the target branch's base-snap-theirs and its HEAD at this branch's base-snap-ours node.

**Semantics**:
- First rewind this branch's HEAD to base-snap-ours, the snapshot corresponding to the co-referent ancestor (the branch name stays the same; the old HEAD position is preserved by a newly created `rebase/<anchor-short>/<n>` branch, naming rules in §4.5)
- Then, taking the target branch's base-snap-theirs as the base point, rebuild each file-object change from after base-snap-theirs up to that branch's HEAD on this branch, one at a time via 4-way merge, producing a series of new file-objects and corresponding Snapshots
- Finally, apply each file-object change this branch originally had from after base-snap-ours up to its original HEAD, again one at a time via 4-way merge, producing a series of new file-objects and corresponding Snapshots, with this branch's HEAD advancing to the end of the new chain

**Execution steps**:

1. Locate the co-referent ancestor (§3.3); collect the ordered Snapshot lists for the target branch from after base-snap-theirs to its HEAD, and for this branch from after base-snap-ours to its original HEAD.
2. Rewind this branch's HEAD to base-snap-ours (keeping the current branch name, creating a new branch that points at the original HEAD).
3. For each step `snap_i` in the target branch's sequence to apply (in chronological order):
   - base = the file-object of `snap_{i-1}` (the first step uses base-object)
   - ours = the file-object at this branch's current landing point
   - theirs = the file-object of `snap_i`
   - Perform a 4-way merge, produce a new file-object, and append a new Snapshot
4. Repeat the logic of step 3 for this branch's own replay sequence from after base-snap-ours, rebuilding this branch's history after the target branch's history.
   - On a **conflict**: pause, output conflict markers, and wait for the user to resolve them before continuing (see §4.7.3)
   - On a **loopback**: prompt the user to choose one of two options — rewind to skip that step (continuing automatically once accepted), or abort and roll back (see §4.7.4)

**History preservation**: a rebase produces a new Snapshot chain and modifies no existing record at the Snapshot layer; the original HEAD position is preserved by the newly created branch and can be inspected at any time.

#### 4.7.2 `lfv merge <file> <FS-ish>`

Insert the history between the target branch's base-snap-theirs and its HEAD at this branch's current HEAD.

**Semantics**:
- Keep this branch's current HEAD where it is (both branch name and position unchanged)
- Taking the co-referent ancestor's base-object as the baseline, apply each file-object change from after the target branch's base-snap-theirs up to its HEAD on top of this branch's HEAD, one at a time via 4-way merge, producing a series of new file-objects and corresponding Snapshots, with this branch's HEAD advancing to the end of the new chain

**Execution steps**:

1. Locate the co-referent ancestor (§3.3); collect the ordered Snapshot list for the target branch from after base-snap-theirs to its HEAD (the sequence to apply).
2. For each step `snap_i` in the sequence to apply (in chronological order):
   - base = the file-object of `snap_{i-1}` (the first step uses base-object)
   - ours = the file-object at this branch's current landing point
   - theirs = the file-object of `snap_i`
   - Perform a 4-way merge, produce a new file-object, and append a new Snapshot
   - On a **conflict** or a **loopback**: handled as in §4.7.3 / §4.7.4

#### 4.7.3 Conflict Handling

When a 4-way merge produces a conflict, LFV pauses the operation and writes conflict markers into the working-area file:

```diff
 <<<<<<< ours (main)
 content from this branch
 =======
 content from the target branch
 >>>>>>> theirs (feature / snap:01HXYZ)
```

Note: every line in this example carries a leading space so that version-control tooling does not treat it as a real conflict. There is no such leading space in actual use.

Conflict markers cannot be written into a binary file: LFV pauses just the same, and the user continues by choosing one side with `--continue --ours` or `--continue --theirs`, or gives up with `--abort`.

After the user has resolved the conflict by editing manually, they run:

```bash
lfv merge --continue   # or lfv rebase --continue
```

LFV takes the resolved working-area content as a new file-object, appends a Snapshot, and continues with the next replay step.

If the user gives up, they run:

```bash
lfv merge --abort   # or lfv rebase --abort
```

LFV restores the branch HEAD to its original pre-operation state and restores the working-area content with it; all intermediate Snapshots already appended are hidden by rolling the branch pointer back (the Snapshots themselves remain in the append-only log but are no longer referenced by any branch).

Passing `<file>` to `--continue` / `--abort` is recommended (e.g. `lfv merge --continue docs/note.md`). When it is omitted, LFV looks for the operation in progress: if exactly one file is in the middle of a merge / rebase, it acts on that file; if several files are, it is an error that asks for `<file>`; if none is, it is an error as well.

#### 4.7.4 Loopback Handling

If some step of the replay produces a file-object that already appears in the current branch's history (violating single-branch object uniqueness, design §4.13), LFV pauses and prompts:

```
warning: step snap:01HXYZ produces content already present in branch 'main'
         (object blake3:abc123...)
options:
  [r] lfv rewind to skip this step and continue rebase/merge
  [a] abort — restore branch to original state
```

- Choosing **rewind**: LFV performs a rewind for the current step (moving the intermediate history into a `detour/<anchor-short>/<n>` branch), then **continues automatically** with the remaining replay steps without asking the user again.
- Choosing **abort**: the abort semantics of §4.7.3; the branch HEAD is restored to its original state.

#### 4.7.5 `lfv merge --pick <file> <snap-id>`

Replay the change of a single snapshot as one step on top of the current branch's HEAD (the equivalent of git cherry-pick): base = the object of that snapshot's parent (empty content when the parent is `null` or the parent's object is `null`), theirs = the object of that snapshot, ours = the object at the current HEAD; one 4-way merge is performed, producing a new file-object and appending a Snapshot. Conflicts and loopbacks are handled as in §4.7.3 / §4.7.4. `<snap-id>` must belong to the same file. A conflict or loopback is likewise continued or rolled back with `lfv merge --continue` / `lfv merge --abort` (§4.7.3/§4.7.4) — no separate `--pick` flag is needed there; the rule for omitting `<file>` is in §4.7.3. The content-deletion workflow in §5.9 depends on this command.

### 4.8 Maintenance

| Command | Description |
| ---- | ---- |
| `lfv gc` | Reclaim objects not referenced by any snapshot. |
| `lfv gc --purge` | Reclaim unreferenced snapshots along with objects not referenced by any snapshot, warning that this is a high-risk operation from which data cannot be recovered. Reachability is defined in design §3.11. |
| `lfv verify` | Verify the integrity of the object store (recompute hashes and compare); recompute each snapshot's `digest` line by line; walk every branch of every file and check single-branch object uniqueness (design §4.13), reporting a violation as a data-integrity error; check that the objects referenced by Tree Objects exist. |
| `lfv rebuild-index` | Discard `index.db` and rebuild it from the source of truth (objects/, snapshots.log, the yaml files) (design §3.10). Does not touch HEAD, branches, tags, or config. |
| `lfv export <file> [--format zip\|tar] -o <out>` | Export the whole history of a file as a self-contained archive for migration. The archive contains that file-id's directory and the File Objects reachable from it, with the same directory structure as `.lfv`. |
| `lfv import <archive>` | Import an archive produced by `lfv export`: merge its file-id directory (`meta.yaml`, `HEAD`, branch/tag tables, `snapshots.log`) and the reachable File Objects into the current repository, keeping the file-id unchanged and merging objects by content hash. Every snapshot `digest` is verified before importing; a corrupted archive is rejected with an error and nothing is partially imported. Errors if that file-id already exists in the current repository (which should not happen normally, as ULIDs are globally unique). **After an import the working tree is not modified and no status-table row is registered** — the file history enters the repository in an "inactive" state and must be restored to the working tree on demand with `lfv revive <file-id>`. Internal execution steps are in design §4.16. |

## 5. Typical Workflows (Spec)

LFV adopts a "**track everything by default**" policy: files under the working directory should normally all be in a tracked state. To that end, relevant commands run a **lazy scan** of the working tree before executing, responding automatically to OS-level create and delete events with no background daemon.

### 5.1 First Use

```bash
cd /path/to/A
lfv init
# Optional: create .lfvignore to statically exclude paths from tracking
echo "node_modules/" >> .lfvignore
echo "*.tmp" >> .lfvignore

lfv status
# -> Lazy scan (design §4.3): trackable new files are auto-tracked (tracking flag set, no Snapshot)
#    All files show the `A` flag and are in modified state in the status table

lfv snap -m "initial snapshot"   # Batch-snapshot all modified files, producing the first Snapshot
```

### 5.2 Day-to-day Editing

```bash
# Edit docs/note.md ...
lfv status docs/note.md         # Check for unsaved changes
lfv snap docs/note.md -m "add chapter 2 outline"
lfv log docs/note.md
```

### 5.3 Diffing

```bash
lfv diff docs/note.md
lfv diff docs/note.md snap:01HXYZ
```

### 5.4 Rewinding to an Old Version (auto-branch)

```bash
lfv log docs/note.md
lfv rewind docs/note.md snap:01HXY0
# -> First automatically creates branch rewind/7RQ2M9KA/1 pointing at main's previous HEAD (history fully preserved)
# -> Then moves main's HEAD to snap:01HXY0, and the working-area content follows
# Subsequent snaps keep landing on main; the old line can be revisited any time with `lfv switch docs/note.md rewind/7RQ2M9KA/1`
```

### 5.5 Rename / Move

```bash
# Scenario A: rename through LFV (recommended, atomic)
lfv mv docs/note.md docs/notes/2026-05/note.md
# -> Appends a rename snapshot; path changes from old to new, object unchanged

# Scenario B: renamed with the OS first, content unchanged — auto-detection is enough
mv docs/note.md docs/notes/2026-05/note.md
lfv status
#   R   file:01HA7BCD   docs/note.md -> docs/notes/2026-05/note.md
#                    auto-detected (identical content hash)
lfv snap                        # Commit all detected changes together

# Scenario C: renamed with the OS and content changed too — auto-detection fails, continue history manually
mv docs/note.md docs/notes/2026-05/note-v2.md
$EDITOR docs/notes/2026-05/note-v2.md
lfv status
#   D   file:01HA7BCD   docs/note.md
#                    file missing on disk; possibly moved
#   A   file:01HA7EEE   docs/notes/2026-05/note-v2.md  [newly tracked]
lfv relink file:01HA7EEE --onto file:01HA7BCD
# -> file:01HA7EEE's content (current path + object) is appended as the next Snapshot on file:01HA7BCD's history
# -> The event type is derived by comparing file:01HA7BCD's last snapshot with the new Snapshot (typically R+M)
# -> file:01HA7EEE is retired (its status-table row is cancelled, snapshots.log is preserved)
```

### 5.6 Delete and Revive

```bash
lfv delete docs/old-note.md     # Deletes the working-tree file and marks the status table as modified
lfv snap -m "remove old note"   # Appends an object = null Snapshot (or leave it to the next no-arg lfv snap batch)
lfv log docs/old-note.md        # History is still fully queryable
lfv revive docs/old-note.md     # main's HEAD returns to the snapshot before the deletion and the content is restored to the working tree;
                                # the delete event is preserved by the automatically created revive/<anchor-short>/1 branch
```

### 5.7 Continuing Old History with a New File of the Same Name

```bash
# Scenario: docs/old-note.md was deleted earlier (that file-id's latest Snapshot has object = null),
#           and a new file has now been created at the same path

lfv status
#   A   file:01HA7EEE...   docs/old-note.md  [newly tracked]
#                       note: this path previously existed as file:01HA7BCD (deleted)
#                       to continue its history instead, run:
#                         lfv relink file:01HA7EEE --onto file:01HA7BCD

# Option A: the new file is simply a new file, unrelated to the old history — snap directly
lfv snap docs/old-note.md -m "new document"

# Option B: the new file continues the old one — resume the old history
lfv relink file:01HA7EEE --onto file:01HA7BCD
# -> file:01HA7EEE's content is appended as the next Snapshot on file:01HA7BCD's history (a revive event)
# -> file:01HA7EEE is retired (its status-table row is cancelled, snapshots.log is preserved, no active trace remains)
```

### 5.8 Global Snapshots (tree plane)

```bash
# After editing all files of the current version, make sure everything is snapped first
lfv snap -m "finish chapter 3"

# Create a tree-snapshot to mark a milestone
lfv snap --tree -m "chapter 3 complete"
lfv snap --tree --tag v1.0 -m "first edition complete"   # tag it at the same time

# View tree history
lfv log --tree
# snap:01HABC  2026-05-30 10:00  "first edition complete" [tree:v1.0]
# snap:01HXYZ  2026-05-17 09:21  "chapter 3 complete"

# Return to a tree-snapshot (the whole working-area content is restored)
lfv rewind tree:v1.0
lfv rewind snap:01HXYZ
```

### 5.9 Content Deletion

When history contains inappropriate content (privacy or security issues, or sensitive versions that are no longer needed), LFV provides a **three-step workflow** for thorough elimination. This section explains the reasoning and how tree snapshots interact with that flow.

> [!warning] `lfv gc --purge` is irreversible. Once it has run, none of the cleaned objects and snapshots can be recovered; run it only when it is clearly needed.

#### 5.9.1 Three-Step Workflow

For each branch containing inappropriate content, execute in order:

1. **Reconstruct history**: `rewind` to the parent snapshot of the one containing inappropriate content, create a new branch. On the new branch, individually `merge --pick` subsequent snapshots from the original branch, editing to remove inappropriate content as needed, forming a new append-only snapshot chain. This entire process operates within existing primitives without violating append-only.
2. **Delete the original branch**: delete the original branch containing the inappropriate content. The snapshot chain on that branch loses all reachable entry points and enters a dangling state.
3. **Physical purge**: run `lfv gc --purge` to delete all dangling snapshots along with their associated File Objects and Tree Objects.

When inappropriate content appears on several branches, apply steps 1–2 to each relevant branch separately, then run a single unified `gc --purge` at the end.

#### 5.9.2 Handling of Tree Snapshots

Tree Snapshots are also bound by the append-only constraint and cannot be modified directly. But once a File Object referenced by a Tree Object has been purged:

- The Tree Snapshot itself **remains** in the append-only log (it can still serve as a timeline node), but the File Object it references is physically gone, so `lfv show` / `lfv diff` queries display `[object missing]`.
- A Tree Object that is no longer referenced by any Tree Snapshot (i.e. the working-directory snapshot it corresponds to was also cleaned through rewind + gc --purge) is deleted along with the other dangling objects.
- **Recommended practice**: before performing a content deletion, move the tree HEAD to a milestone free of the inappropriate content with `lfv rewind tree:<tag>`, to avoid leaving broken references on the tree plane. If a broken reference is created inadvertently, `lfv verify` reports the Tree Object pointing at a non-existent File Object.

Dangling snapshots and File/Tree Objects are retained by default (the user may want to recover them later); only `gc --purge` performs true physical deletion.

### 5.10 Cross-device Sync and Collaboration

Simply copy or sync the entire working directory (including `.lfv`) to another device through a NAS or cloud drive. LFV itself **does not resolve concurrent write conflicts** — it is the sync tool's responsibility to ensure `.lfv` is not modified simultaneously.

For collaborative work with repositories in other locations, remote repositories can be used. The details are omitted here; the specification will be settled later.

## 6. Technology Choices Design

- **Language**: Rust 2024 edition.
- **CLI framework**: `clap` v4, defining the command tree in derive style.
- **Error handling**: `thiserror` (error types at library level) + `anyhow` (top-level wrap-up in the CLI).
- **Hashing**: `blake3` (fast and strong enough).
- **Identifiers**: `ulid` (generates the ULIDs used for file-ids and snap-ids).
- **Compression**: `zstd` level 3; defaults `min_bytes` 4 KiB, `max_bytes` 16 MiB, `reject_if_larger: true` (details in design §3.2).
- **Metadata serialization**: `serde` + strict YAML (human-readable config/metadata; format constraints in §3.5) + `serde_json` (the line format of snapshots.log). All config and metadata files under `.lfv` uniformly use the `.yaml` extension.
- **Index storage**: `rusqlite` (embedded SQLite, a single `index.db` file).
- **Text diff and three-way merge**: `diffy` (Myers diff, unified format, `merge` three-way merge with conflict markers).
- **Time**: `jiff`.
- **Logging**: `tracing` + `tracing-subscriber`, with the level controlled by `-v/-vv` on the CLI.
- **Testing**: `assert_cmd` + `predicates` + `tempfile` for integration tests; unit tests live next to the code in their module.

> Selection principle: prefer libraries already widely proven in the Rust ecosystem and avoid unmaintained crates. All dependencies are reviewed once per quarter before 1.0.

## 7. Roadmap (rough)

- **v0.1**: `init` / `config` / `track` / `untrack` / `snap` / `status` / `log` / `show` / `list` / `rebuild-index`; plus `branches` / `switch` / `rewind` — the loopback hint of `lfv snap` tells the user to run `rewind`, so those three must land together with `snap`.
- **v0.2**: `diff` / `mv` / `delete` / `revive` / `relink` / `branch-rename` / `branch-delete`.
- **v0.3**: the `tag` family, `export` / `import`, `gc`, `verify`.
- **v0.4**: `merge` / `rebase` / `merge --pick`, conflict handling.
- **v0.5**: tree plane (`snap --tree` / `log --tree` / `rewind <TS-ish>` / `tag --tree`), performance tuning.
- **v1.0**: stable CLI semantics, complete documentation, cross-platform CI passing.
- **v1.1**: remote repositories, completing the collaboration work.
