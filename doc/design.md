# LFV — Lightweight File Versioning

> This is the English design document for the LFV project. For the Chinese version, see `design.zh-cn.md`.

---

## 1. Background and Motivation

`git` is an excellent version-control tool, but its design target is **repository-scoped** version management — best suited for a logically cohesive collection of source files where a single commit can span multiple files and branches record the holistic evolution of an entire working tree.

Yet daily work also contains another archetypal scenario with the following characteristics:

- Each file is **logically independent**, with no coupling to any other;
- Each file has **its own line of history**, unrelated to other files;
- There is no semantic need to "commit multiple files at once";
- Users care about the evolution, comparison, and rollback of **one specific file** along a timeline.

Typical examples:

1. **NAS file sync systems**: each backed-up file has its own history, but cross-file atomic commits are unnecessary.
2. **One-note-per-file repositories**: each note evolves independently; changes to note A should not "pollute" note B's history.
3. **Design drafts / contracts / single-document archives**: each document tracked individually.
4. **Configuration file directories**: each config file evolves independently, without affecting others.

Existing approaches to this scenario are generally unsatisfying:

- **Simple backup**: copies old versions only, with no way to record the purpose of each version;
- **Retain the last N versions**: lacks semantic information; no diff, no annotation;
- **Whole-directory git repository**: mixes the evolution of unrelated files, buries individual file history, and violates the "file-centric" mental model;
- **Cloud drive version history**: typically unusable offline, and lacks branching, comparison, or portability.

## 2. Project Goals

LFV (Lightweight File Versioning) aims to:

1. Create a `.lfv` directory inside a designated working directory, serving as the **local version repository** for all tracked files in that directory tree.
2. Provide an `lfv` CLI so users can, intuitively and analogously to `git`:
   - Selectively track a file;
   - Create annotated snapshots of that file;
   - View the list of historical snapshots for that file;
   - Diff any two snapshots (or a snapshot against the current content);
   - Rewind to a historical snapshot — **automatically creating a new branch** on rewind, never destroying existing history.
3. Each file has **independent**: history chain, branch set, and tag set.
4. Provide good portability: the `.lfv` directory can be copied or synced together with the working directory and behaves consistently after migration.
5. Compact data storage: content-hash deduplication and compression to avoid storage bloat from naive backups.

## 3. Non-Goals (explicit exclusions)

In order to stay "lightweight", the following are **out of scope**:

- **Multi-file atomic commits**: each snapshot targets a single file; there is no "commit multiple files at once" semantics.
- **Distributed collaboration**: no push / pull / remote / merge or other multi-user coordination semantics. Sharing `.lfv` via external sync (e.g. NAS, cloud drive) is fine, but concurrent conflicts are not resolved by LFV.
- **Full merge algorithm**: no automatic 3-way merge for two branches of the same file; the user decides which branch to keep.
- **Replacing git**: source-code engineering scenarios should continue using git; LFV serves only the "file-centric" scenario.
- **Graphical interface**: v1.0 provides CLI only; a GUI is listed as a possible future extension.
- **Staging area**: there is no staging step — the working area is directly the subject of each snapshot.

## 4. Core Concepts

| Term | Description |
| --- | --- |
| Repository | The `.lfv` directory under the working directory; holds metadata and object storage for all tracked files. |
| Working Tree | The working directory itself (excluding `.lfv`); where users actually operate on files. |
| Tracked File | A file brought under repository management by `lfv track`. Identified internally by a `file-id` (ULID) that is **fully decoupled from its path**; the file's location in the working tree is a field on each Snapshot and can evolve over history. |
| File Status | For paths already in LFV's view, status is `unmodified`, `modified`, or `untracked` (see §6.6). |
| Object | A content-addressed storage unit for file content; deduplicated by content hash — different files may share the same object. |
| Snapshot | An "event record" for a tracked file at a point in time: content pointer + path + metadata (message, timestamp, author, parent snapshot). A single Snapshot covers content changes, renames/moves, and deletions. After a file is successfully snapshotted, its status in the status table becomes `unmodified`. |
| Branch | A chain of snapshots for a tracked file; the default branch is `main`. Branch namespaces are independent per file. |
| HEAD | The current branch and latest snapshot pointer for a tracked file. |
| Tag | A human-readable name for a snapshot (optional), used to stably reference a specific version. |
| Action | Actions change the state of a file. Available actions include `track`, `snap`, and `untrack`, plus two actions with no corresponding command: `modify` (achieved by the user editing the file) and `auto-track` / `auto-delete` (applied automatically by LFV during scanning in response to OS file create/delete events; see §6). |
| LFV-visible | Files remaining in the working directory after `.lfvignore` directory pruning. |

> [!Note]
> **All of the above concepts are scoped to a single file** — this is the most fundamental difference between LFV and git.

### 4.1 File Identity: file-id Decoupled from Path

`file-id` is the stable internal identity of a tracked file within the repository:

- A unique ULID is assigned **at first track**, e.g. `f_01HXYZABC...`;
- It has **no derivation relationship** with the file path and does not change on rename/move;
- Users locate files at the CLI layer by **current path**; the CLI resolves this via the `fullpath → file_id` index in the status table;
- The "current path" of a tracked file is **not** stored in `meta.yaml` but maintained by the status table, which is **derived and updated** from events on the Snapshot chain — so that the repository always has a single source of truth (the Snapshot chain) and mutable state can be reconstructed.

### 4.2 Storage Objects: Object and Snapshot

There are **only two true "storage objects"** in an LFV repository:

| Storage Object | Contents | Addressing | Immutability |
| --- | --- | --- | --- |
| **Object** | Raw file bytes (compressed) | Content-addressed: `blake3(content)` | Fully immutable (write-once) |
| **Snapshot** | Event record: path, object pointer (nullable), message, author, timestamp, parent snapshot id, digest | Identifier-addressed: `snap_<ULID>` | Append-only, never rewritten |

Everything else (HEAD, branches, tags, file status, config) is **mutable state / index** and is not a "storage object." Mutable state can be reconstructed if corrupted (as long as Objects and Snapshots are intact); storage objects are immutable — the repository's source of truth.

About Snapshots:

- The entire file lifecycle is the Snapshot chain.
- A Snapshot carries the `path` field; the path is not the file's identity.
- **Rename/move** is also an immutable history event — structurally identical to "add", "content change", and "delete", distinguishable only by comparing fields with the parent snapshot.
  Rename = appending a new Snapshot; delete = appending a Snapshot with `object=null`.
- Each Snapshot has **exactly one parent pointer**, forming a directed tree (forest). This differs from Git's DAG topology: once branches diverge, they evolve independently and there is no topological merge point.
- In rendering and visualization layers, however, snapshots with the same `file-id` can be treated as one group.

**There is no tree object** — the biggest structural difference from Git. Git's tree describes "the directory listing of a commit," but LFV's snapshot naturally corresponds to a single file; a Snapshot points directly to an Object with no intermediate layer.

### 4.3 Mutable Index: Status Table, Branches, HEAD, Tags

The mutable index lives in `index.db`, recording working-directory file status and branch/tag information for each file. It is a reconstructable cache of current state and is not part of the immutable history objects. `index.db` contains the following index tables:

- **file_states** (status table): core fields include `fullpath`, `status` (`untracked` / `modified` / `unmodified`), and `file_id` (null when `untracked`).
- **scan_meta** (scan metadata): `last_completed_at`, mtimes of `.lfvignore` and `config.yaml` from the last scan, etc., used for incremental scans and rule invalidation (see §6.8).
- **branches**: per-file branch pointer table.
- **tags**: per-file tag table.
- **head**: per-file current branch and latest snapshot pointer.

> [!note] About `file_states`
> - Normally, it records only LFV-visible files that currently exist in the working tree (`stat` succeeds and the path does not match `.lfvignore`);
> - `untracked` rows correspond to paths on the dynamic `untracked` list in `config.yaml` that also exist on disk;
> - if a path disappears from disk, its row is deleted (`config.yaml` may keep the list entry);
> - **Exception**: a tracked file that disappeared from disk but is waiting for `lfv snap` to record the current FS state remains as `modified`; `status` renders it as `D` (§6.5).
> - For tracked files that still exist on disk, `fullpath → file_id` resolves CLI paths; after the file disappears, it can still be addressed by `file-id`.

## 5. Repository Layout

```
A/                                   # Working directory
├── docs/note.md                     # Example tracked file
├── photos/2025/sunset.jpg
└── .lfv/
    ├── config.yaml                  # Repository-level config (strict YAML)
    ├── HEAD                         # Global placeholder (reserved, mainly for compatibility)
    ├── index.db                     # Status, branches, tags, head index (SQLite)
    ├── objects/                     # Content-addressed object store
    │   ├── ab/
    │   │   ├── cdef0123...zstd      # zstd-compressed object
    │   │   └── cdef0123...raw       # raw object (too small/large/incompressible)
    │   └── ...
    ├── files/                       # Per-tracked-file metadata
    │   └── <file-id>/
    │       ├── meta.yaml            # File-level metadata (creation time, optional attributes)
    │       ├── branches.yaml        # Branch table for this file
    │       ├── tags.yaml            # Tag table for this file
    │       └── snapshots.log        # Append-only snapshot records (JSON Lines)
    └── logs/                        # CLI operation logs (optional, for debugging)
```

### 5.1 Object Store

- Content-addressed: object identity = `blake3(raw bytes)`; the hash prefix in the bucket path is derived from raw content, independent of whether the object is compressed;
- Bucketing: first 2 hex characters used as directory name to avoid excessive files in a single directory;
- Compression: raw bytes are compressed with `zstd` (default level **3**) and stored with a `.zstd` extension; decompression is performed in memory on read;
- Raw storage (`.raw` extension): store uncompressed when **any** of the following holds:
  - **floor**: `size < min_bytes` (default **4 KiB**, 4096 bytes) — too small; frame overhead may make compressed output larger;
  - **ceiling**: `size >= max_bytes` (default **16 MiB**, 16777216 bytes) — avoid memory spikes from reading entire large files;
  - **ineffective compression**: after attempting compression, `compressed_len >= original_len` (`reject_if_larger`, default **true**) — no benefit for already-compressed binaries, etc.;
- Cross-file sharing: identical content is stored as a single object, saving space.

#### 5.1.1 Default Compression Config (`config.yaml`)

```yaml
compression:
  enabled: true
  algorithm: zstd
  level: 3              # zstd 1..=22
  min_bytes: 4096       # floor: size < min → .raw
  max_bytes: 16777216   # ceiling: size >= max → .raw
  reject_if_larger: true
```

### 5.2 Snapshot Record Format

`snapshots.log` is one JSON object per line (JSON Lines):

```json
{
  "id": "snap_01HXYZ...",
  "branch": "main",
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

Field descriptions:

- `id`: a monotonically sortable, human-readable snapshot identifier using [ULID](https://github.com/ulid/spec) with a `snap_` prefix. ULIDs embed a timestamp prefix and random suffix, making time-ordered sorting in `log` output easy and paste-friendly in a terminal.
- `parent`: parent snapshot id; `null` for the first snapshot on a branch.
- `path`: **the file's relative path in the working tree at the time this snapshot was taken**. A rename/move event is expressed as this field differing from `parent.path`; a deletion event retains the pre-deletion path (for readable `log` output).
- `object`: the blake3 hash of the referenced Object (prefixed with `blake3:` to allow future algorithm migration); **`null` when this snapshot is a deletion marker**.
- `size`: byte count of the corresponding Object; `0` for deletion events.
- `digest`: **a `blake3` hash of all fields in this snapshot record except `digest` itself, after canonical serialization**, used for tamper detection. `lfv verify` recomputes and checks this for every line.
- All other fields are self-explanatory.

Designed to be append-only, facilitating incremental backup and auditing.

#### 5.2.1 Deriving the Event Type

LFV does not store an explicit "event type" field in a Snapshot. Instead, the event type is **derived** from the `(parent, path, object)` triple relative to the parent snapshot:

| Parent path | Parent object | Current path | Current object | Event type | Rendered in `log` |
| --- | --- | --- | --- | --- | --- |
| — (first) | — | P | O | **add** (first snap) | `A  P` |
| P0 | O0 | P0 | O1 | **modify** | `M  P0` |
| P0 | O0 | P1 | O0 | **rename** | `R  P0 -> P1` |
| P0 | O0 | P1 | O1 | **rename + modify** | `R+M  P0 -> P1` |
| P0 | O0 | P0 | **null** | **delete** | `D  P0` |
| P0 | **null** | P1 | O1 | **revive** | `+  P1` |

> The `track` action only registers a `file-id` and the tracking flag; **it does not produce a Snapshot**. The first entry on the Snapshot chain is produced explicitly by the user running `lfv snap`, and its event type is derived as `add` (`parent=null`, `object≠null`). `track` and the first `snap` are two distinct steps; the same applies to auto-track.

> The benefit of not storing an explicit event type: every Snapshot is a complete "what the file looks like at this moment" record; adding new event types in the future requires no new fields or migration of old records. The trade-off: event type must be derived on the read side, so the rendering logic must have unit test coverage for all combinations.

> **Why Snapshot ids use ULID rather than content hash**:
> Git uses the content hash of a commit as its id, giving natural tamper-resistance but non-human-readable, non-time-sortable ids. LFV uses ULID because:
> 1. In a single-file scenario, users frequently need to visually select snapshots by time from `log` output; ULIDs are far more readable than raw hashes;
> 2. Tamper-resistance is delegated to the dedicated `digest` field, giving clean semantics and enabling targeted verification by `lfv verify`;
> 3. Decoupling id from digest means adding or removing metadata fields in the future will not cause id drift.

## 6. File Lifecycle Events

This section describes all actions that change a file's tracking state — including automatic scan responses by LFV and explicit commands run by the user.

### 6.1 Tracking Policy: Track-by-Default

LFV adopts a **track-by-default** policy: files in the working directory should, under normal circumstances, all be in a tracked state. To this end, relevant commands run a **lazy working-tree scan** before executing (algorithm in §6.8), automatically responding to OS-level create and delete events — no background daemon required.

### 6.2 Auto-track (New Files)

When a scan finds a path that is **LFV-visible** and not yet in the status table:

- If it matches `.lfvignore` -> **ignore** it (no registration; §6.6);
- If it is on the dynamic `untracked` list in `config.yaml` -> register it as `untracked` (§6.6);
- Otherwise -> treat it as an OS-created file, auto-`track` it, assign a `file-id`, and set status to `modified`.

No Snapshot is appended immediately; the first Snapshot is written by a later `lfv snap`.

Auto-track is triggered by: `lfv status` (applied immediately during scan), `lfv track` (with no arguments), and `lfv snap` (before batch-snapshotting with no arguments).

**Cancelling tracking**: `lfv untrack <file>` writes the path into the dynamic `untracked` list in `config.yaml`; when the path exists on disk, it synchronously records `untracked` in the status table. Files with existing history are guaranteed not to produce new Snapshots. Auto-tracked files with no history only have their tracking cache removed, ensuring no Snapshot is appended.

When a path is in config `untracked` and exists on disk, scans will not auto-track it. If it disappears from disk, the status-table row is deleted (§6.8); when it reappears, it is registered as `untracked` again per §6.2.

**Friendly hint for same-path new files**: if a path previously had a deleted `file-id` (i.e. a deletion marker was appended for that path), and auto-track assigns a new `file-id` at the same path, `lfv status` will append a hint below that file's entry:

```
  M   f_01HA7EEE...   docs/old-note.md  [newly tracked]
                      note: this path previously existed as f_01HA7BCD (deleted)
                      to continue its history instead, run:
                        lfv relink f_01HA7EEE --onto f_01HA7BCD
```

This hint appears only when auto-track has assigned a new `file-id`; once the new `file-id` records its first Snapshot, it stands as an independent file and the hint disappears.

For `untracked` files that still have history, because their latest Snapshot still points to an Object, running `lfv track <file>` again first removes the entry from the dynamic `untracked` list in `config.yaml`, then resumes tracking under the original `file-id` and updates the corresponding status-table row to tracked state as a cache. Note that this differs from tracking a deleted file: after delete, `lfv track <file>` creates a new `file-id` by default unless the user explicitly runs `lfv revive`.

### 6.3 Rename / Move (`lfv mv`)

**`lfv mv` performs path operations only**: both `<src>` and `<dst>` accept only paths, not `file-id`s. It is used when the file still exists on disk (or was just moved by the OS). Accepting a `file-id` as `<dst>` is intentionally disallowed — it would cause mental confusion, as users might assume the surviving `file-id` is the one on the `<dst>` side. To splice history onto another `file-id`, use `lfv relink` (see §6.4).

- `lfv mv <src> <dst>`: migrates the tracked file at `src` to the `dst` path. The rename is staged; the resulting Snapshot is recorded by a subsequent `lfv snap`.
  - If `src` still exists in the working tree and `dst` does not, the CLI first moves the file to `dst` on disk, then stages the rename.
  - If `src` no longer exists (already moved by the OS or editor) and `dst` already contains the file, `lfv mv` only stages the rename registration without touching the disk.
  - The new Snapshot's `object` is determined by the hash of `dst`'s current content at snap time: if it matches the parent snapshot's object, the event renders as pure `R`; otherwise as `R+M`.
  - The default `message` is `rename: <old-path> -> <new-path>`.
- **Auto-detection vs. manual registration**:
  - Auto-detection applies only when content hashes are **exactly equal** (`rename.autodetect`, enabled by default). When a match is found, `lfv status` displays an `R` line and automatically pairs the files.
  - If the OS moves a file **and** its content changes, auto-detection fails — `lfv status` will display both a `D` line (old `file-id` missing from its original path) and an `A` line (new file at the new path with a new `file-id`). The user reviews and explicitly declares history continuation with `lfv relink <new-file> --onto <old-file>`.
  - Auto-detection can also be disabled (useful for bulk rename + edit scenarios to avoid mispairing).
- History display: `lfv log <file>` renders rename events as "`R` old-path -> new-path", alongside `A`(add) / `M`(modify) / `D`(delete) / `R+M`(rename+modify) (see §5.2.1).

### 6.4 History Continuation (`lfv relink`)

`lfv relink <f_src> --onto <f_dst>` splices the history of `f_src` onto the end of `f_dst`'s history, declaring that "f_src is the continuation of f_dst." At the same time, the on-disk file previously associated with f_src becomes bound to f_dst going forward.

A typical use case: a rename the scanner cannot auto-pair — the user changed both path and content at the OS level, so LFV sees an independent `D` event (f_dst, old path disappeared) and an `A` event (f_src, new file at the new path). The user manually declares "f_src is f_dst continued."

- **f_src**: status `A`/`M`, a live file that exists on disk (new `file-id`, still being tracked)
- **f_dst**: status `D`, a file that has disappeared from disk (old `file-id`, awaiting continuation)
- **The surviving `file-id` is the `--onto` side (f_dst)**, consistent with the preposition's direction

After execution, f_src's current path and content are appended as the next Snapshot on f_dst's history; f_src's `file_states` tracking row is cancelled and it produces no further snapshots. **f_src's `files/<f_src>/` directory and `snapshots.log` are fully preserved** (append-only; not deleted). f_src becomes a "retired" `file-id`: `lfv log f_src` remains valid.

**Case 1: f_src has no snapshot history**

f_src has only a tracking record and no Snapshots yet. After execution: the file at f_src's current path is associated with f_dst's `file-id`. The event type is derived per §5.2.1 by comparing f_dst's last snapshot with the current path/object (typically `R+M` or `R`).

**Case 2: f_src already has snapshot history**

f_src has produced a chain of Snapshots (`snap_A1 -> snap_A2 -> ... -> snap_An`). After execution, the entire chain is copied and appended to f_dst's history:

- Each snapshot is assigned a new ULID (`snap_B1 ... snap_Bn`); `digest` is recomputed for the new fields.
- `snap_B1.parent` = f_dst's latest snapshot; `snap_Bx.parent` = `snap_B(x-1)`.
- `snap_B1`'s event type is derived by comparing f_dst's last snapshot with `snap_B1`'s path/object; `snap_B2` onward is consistent with f_src's internal derivation and is unaffected.
- The event type at the seam may appear as a cross-path jump — this is the expected trade-off of explicitly declaring history continuation.
- Likewise, the on-disk file previously associated with f_src becomes bound to f_dst going forward.

### 6.5 Auto-delete (OS-deleted Files)

When a scan finds that a **tracked file's** path has disappeared from the working tree, and there is no evidence of a rename (i.e. rename auto-detection could not pair it), the file is treated as "OS-deleted."

LFV does **not** immediately append a deletion-marker Snapshot; instead it marks the file as `modified`. Because the current path does not exist in the filesystem, `lfv status` renders it as `D`.

The next `lfv snap` records the state update. Subsequent processing is described in §6.6.

This design gives the user a window before `lfv snap`: while a file is still in `D` status, they can still run `lfv mv f_old <new-path>` to reclassify it as a rename, avoiding accidental deletion.


### 6.6 `lfv delete`

`lfv delete <file>`: sets the file to `modified` in the status table and deletes it from disk. On the next `lfv snap`, LFV checks the target file's current filesystem state: if the file does not exist, it appends a Snapshot with `object = null` and removes the corresponding `file-id` from the current-path index in the status table.

After this, the `file-id` can no longer be resolved via the current path, but the last known path is preserved in that Snapshot; `lfv list --deleted` can still display it by path.

**Complete history is preserved**: the file can still be queried via `lfv log` / `lfv show` / `lfv diff`; to "revive" it, use `lfv revive` (see §7.2) or `lfv rewind <snap>` — both automatically create a new branch (without disturbing the existing deletion event).

### 6.7 `.lfvignore` and `config.yaml` (Two Layers)

Track-by-default would otherwise include temporary files and build artifacts. `.lfvignore` and `config.yaml` are **not** the same kind of "exclusion list":

| | `.lfvignore` | `config.yaml` (dynamic `untracked`) |
| --- | --- | --- |
| Nature | Static, repo-wide, **highest priority** | Dynamic policy via `lfv untrack` / `lfv track` |
| Status table | **Not stored** in `file_states` | Stored as `untracked` when it **exists** on disk; when it disappears from disk, the status-table row is **deleted** (`config.yaml` list entry may remain) |
| Working-tree scan | **Invisible**: pruned/skipped; no OS create/delete detection | **Visible**: participates in §6.8; status-table rows are maintained only for paths that exist on disk |
| `lfv track <file>` | **Error** if matched; edit `.lfvignore` | Remove from `untracked` list and track |
| `lfv untrack <file>` | **Error** if matched | Write config; if it exists on disk, record `untracked` |
| `lfv status --include-untracked` | **Never listed** | **Only source** (`untracked` rows in the status table, all from config) |

Additional rules:

- `.lfv/` is always implicitly treated like `.lfvignore` and is not stored in the status table;
- paths matching `.lfvignore` **do not exist** for LFV: no auto-track, no untrack listing, not in `--include-untracked`;
- on `rebuild-index`, `untracked` rows = dynamic `config.yaml` list ∩ LFV-visible paths ∩ paths that **exist on disk**.

### 6.8 Working-Tree Scan (Lazy Scan)

Several commands (`lfv status`, `lfv track`, `lfv snap`, etc.) trigger a working-tree scan beforehand, comparing disk against `index.db`, updating `file_states`, and driving the automatic actions in §6.2–§6.6.

#### 6.8.1 Scan Actions

Definitions:

- Dynamic untracked: a path is in the dynamic `untracked` list in `config.yaml`.
- Trackable candidate: a path is not in the dynamic `untracked` list in `config.yaml`.
- Status update:
  - If LFV-visible and dynamic untracked: set status to `untracked`;
  - If LFV-visible and already has a tracked status row: compare mtime/size -> compare hash -> set `modified` or `unmodified`; the chain does not necessarily need to run to completion before a result is known;
  - If LFV-visible, unregistered, and a trackable candidate: auto-track (§6.2);
  - If LFV-invisible: delete the corresponding status-table row, keep `config.yaml` and historical Snapshots unchanged, and do not produce `D`;
- Dynamic untracked: a path in the dynamic `untracked` list in `config.yaml`.

| Layer | Scope | Behavior |
| --- | --- | --- |
| **A. Registered paths** | Rows in `file_states` with an existing `fullpath` | For each row, first decide whether LFV visibility must be rechecked based on `.lfvignore` mtime. If invisible, delete the status-table row; if visible, update status. Cost O(registered paths). |
| **B. Discover new paths** | LFV-visible paths seen during traversal that are not yet registered | Dynamic untracked files are set to `untracked`; trackable candidates are auto-tracked (§6.2). |

All scan actions update the status table. In addition, `lfv track` and `lfv untrack` update both `config.yaml` and `file_states` (`untracked`).

#### 6.8.2 Incremental Scan and Invalidation

`scan_meta` records the last scan completion time and rule-file mtimes. Scans are **incremental by default** to avoid fully recursing the working tree on every command:

- **Invalidate** (any triggers a controlled full walk for layer B):
  - `scan_meta` is empty (first scan or after `rebuild-index`);
  - `.lfvignore` mtime is newer than recorded (walk/prune boundaries change; **does not** write ignore paths into the status table);
  - `config.yaml` mtime is newer (reconcile the dynamic `untracked` list with LFV-visible paths: register `untracked` rows for paths that exist, delete rows for paths that do not);
  - user runs `lfv status --refresh` (or equivalent forced refresh).
- **Otherwise (incremental)**:
  - **Directories**: if dir mtime ≤ `last_completed_at` and the directory is already registered as scanned → do not descend; combined with `.lfvignore` **directory pruning** (pruned trees are invisible to LFV);
  - **Files**: if already in `file_states` and file mtime ≤ `last_completed_at` → skip layer-B "new path" handling (layer A still `stat`s).
- **Scan end**: update `last_completed_at` (recommended: scan start time) and `.lfvignore` / `config.yaml` mtimes.

> **Note**: mtime-based incrementality may miss changes when copies do not preserve timestamps or on coarse-grained filesystems; use `--refresh` as a fallback.

#### 6.8.3 When Scans Run

| Command / scenario | Scan |
| --- | --- |
| `lfv status` | Default incremental scan (§6.8.2) |
| `lfv status --refresh` | Scan after forced invalidation |
| `lfv track` (no args), `lfv snap` (no args) | Scan before batch track / snap |
| `lfv status --include-untracked` | Does **not** add or alter scanning; output only (§7.3.2) |

## 7. CLI Commands

> Convention: `<file>` refers to a relative or absolute path of a file within the working directory; the CLI normalizes all paths internally to be relative to the repository root.

### 7.1 Repository Management

| Command | Description |
| --- | --- |
| `lfv init` | Create `.lfv/` in the current directory. Errors if it already exists. |
| `lfv config <key> [value]` | Read or write repository configuration (e.g. `user.name`). |

### 7.2 File Tracking

| Command | Description |
| --- | --- |
| `lfv track [<file>]` | Add a file to tracking. When `<file>` is given, errors if the path matches `.lfvignore`; otherwise removes it from the `untracked` list in `config.yaml`. If the path was previously untracked (not deleted), the original `file-id` is reused and the status table is updated to `modified` (awaiting the first `lfv snap` if there is no history). With no argument, scans and tracks all trackable files not yet in the status table. |
| `lfv untrack <file>` | Stop tracking: errors if LFV-invisible; if visible, updates the dynamic `untracked` list in `config.yaml`; if it exists on disk, records `untracked` in the status table, otherwise config only. History is preserved; re-track via `lfv track` / `lfv revive`. |
| `lfv mv <old> <new>` | Migrates the tracked file at `<old>` path to `<new>` path. Both arguments accept only paths, not `file-id`s. The rename is staged; a subsequent `lfv snap` records the Snapshot. Whether an actual file move is performed in the working tree is governed by §6.3. |
| `lfv relink <f_src> --onto <f_dst>` | Splices f_src's history onto f_dst, declaring "f_src is the continuation of f_dst." f_src's snapshots (if any) are appended to f_dst's history with new ULIDs; f_src's `file_states` tracking row is cancelled but its `snapshots.log` is fully preserved. See §6.4. |
| `lfv delete <file>` | Sets the file to `modified` in the status table and deletes it from disk. On a later `lfv snap`, if it is `modified` and absent from disk, LFV appends an `object = null` Snapshot, removes it from the `untracked` list in `config.yaml`, and updates the status-table cache. Its history remains complete and it can be revived at any time. |
| `lfv revive <ref>` | Revive a deleted file. `<ref>` may be a `file-id`, the last known path, or a specific Snapshot id. Automatically creates a new branch (`revive/<...>`) and restores the content to the working tree from the selected snapshot. |
| `lfv list [--deleted]` | List all tracked files with their current branch and latest snapshot summary. By default shows only active files; `--deleted` also lists files with a deletion marker. |

### 7.3 Status and Snapshots

| Command | Description |
| --- | --- |
| `lfv status [<file>]` | **Without `<file>`: list all `modified` tracked files.** Runs a lazy scan per §6.8 first. Default output shows tracked changes only, each line with `file-id` (`f_*`). `--include-untracked`: see §7.3.2. `--refresh`: force a full scan-cache refresh. With `<file>`: that file only. |
| `lfv snap [<file>] [-m <msg>]` | Create a new snapshot for a file. **Without `<file>`: batch-snapshot all `modified` tracked files.** Refuses if working content is unchanged (unless `--allow-empty`). |
| `lfv log <file>` | List the snapshot history for a file. Supports `--branch <name>`, `--graph`, `--limit N`. |
| `lfv show <file> <snap>` | Output metadata for a specific snapshot; `--content` outputs the content; `--out <path>` exports it. |

Notes:
- For `modified` files in `lfv status`: deleted files show as `D`; files whose path differs from the last snapshot show as `R`; files with no snapshot yet show as `A`; all others show as `M`; files with both content and path changes show as `R+M`.
- During `lfv snap`: each target modified file is checked against the current filesystem state. If the file exists, LFV writes or reuses an Object and appends a content Snapshot; if it does not exist, LFV appends a Snapshot with `object = null`; `unmodified` files are skipped. A lazy scan per §6.8 runs first; when `<file>` is specified, only that file is processed.

#### 7.3.1 `lfv status` Output Format (Tracked Files)

`lfv status` output looks like:

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

**Status flags:**

- `A` = add (newly tracked, no snapshot yet)
- `M` = modified (includes newly tracked files not yet snapshotted)
- `R` = rename (auto-detected pending rename; explicit `lfv mv` appends a Snapshot immediately)
- `D` = suspected delete (**tracked** file missing from disk; confirmed as deletion marker on next `lfv snap`)
- `~` = dynamic untracked (`status = untracked`, config policy + file exists on disk)

**The `file-id` column:**

Displays `f_*` (ULID with `f_` prefix), abbreviated to the first 10 characters by default; `--long` shows the full length. Assigned at track time and persists through the file's entire lifecycle — it is the only stable reference token at the CLI level.

**Any command that accepts `<file>` also accepts either a path or `f_*` as its argument:**

- `lfv relink f_01HA7EEE --onto f_01HA7BCD` — splices f_01HA7EEE (new `file-id`, exists on disk) onto f_01HA7BCD (old `file-id`, disappeared); f_01HA7BCD continues as the active `file-id`, f_01HA7EEE is retired but preserved.
- `lfv mv docs/old-note.md docs/notes/new.md` — path rename; stages a rename snapshot.
- `lfv delete f_01HA7DEF` — can explicitly register a deletion by `file-id` even when the file is no longer in the working tree.

#### 7.3.2 `--include-untracked` (Display Dynamic Untracked)

Separate from the working-tree scan in §6.8: this flag **only changes output** — no extra full-tree walk.

- **Single source**: rows with `file_states.status = untracked` (all already confirmed by scanning to exist on disk; §6.8.1).
- LFV-visible only: paths matching `.lfvignore` never appear (§6.6). `.lfvignore` is a static exclusion rule; `lfv status` does not list LFV-invisible files.
- After the tracked-files block in §7.3.1, append an **Untracked** section as `~` lines (no `file-id`); `stat` for size. Paths that disappeared from disk have no status-table row, so there is no `~` line.

See the §7.3.1 example; `~` meaning is in that section's status flags.

### 7.4 Diffing

| Command | Description |
| --- | --- |
| `lfv diff <file>` | Working tree vs. latest snapshot. |
| `lfv diff <file> <snap>` | Working tree vs. a specific snapshot. |
| `lfv diff <file> <snapA> <snapB>` | Between two snapshots. |

Text files use line-based diff (default 3-line context); binary files show only metadata differences (size, hash).

### 7.5 Rewind and Branches

| Command | Description |
| --- | --- |
| `lfv rewind <file> <snap>` | Restore the working-area file content to the specified snapshot. **Automatically creates a new branch** (naming pattern `rewind/<snap-short>/<n>`) and moves HEAD to the new branch. |
| `lfv branches <file>` | List all branches for this file. |
| `lfv switch <file> <branch>` | Switch the file's current branch (also updates working-area content to the head snapshot of that branch). |
| `lfv branch-rename <file> <old> <new>` | Rename a branch. |
| `lfv branch-delete <file> <branch>` | Delete a branch (only deletes the pointer; objects are retained in case of sharing). |

### 7.6 Tags

| Command | Description |
| --- | --- |
| `lfv tag <file> <snap> <name>` | Tag a snapshot with a name. |
| `lfv tags <file>` | List all tags for this file. |
| `lfv tag-delete <file> <name>` | Delete a tag. |

### 7.7 Maintenance

| Command | Description |
| --- | --- |
| `lfv gc` | Reclaim objects not referenced by any snapshot. |
| `lfv verify` | Verify object store integrity (recompute hashes and compare). |
| `lfv export <file> [--format zip\|tar] -o <out>` | Export all history for a file as a self-contained archive for migration. |

## 8. Typical Workflows

### 8.1 First Use

```bash
cd /path/to/A
lfv init
# Optional: create .lfvignore to statically exclude paths
echo "node_modules/" >> .lfvignore
echo "*.tmp" >> .lfvignore

lfv status
# -> Lazy scan (§6.8): trackable new files are auto-tracked (tracking flag set, no Snapshot yet).
#    All files show the `A` flag and have `modified` status in the status table.

lfv snap -m "initial snapshot"   # Batch-snapshot all modified files; produces first Snapshot
```

### 8.2 Day-to-day Editing

```bash
# Edit docs/note.md ...
lfv status docs/note.md         # Check for unsaved changes
lfv snap docs/note.md -m "add chapter 2 outline"
lfv log docs/note.md
```

### 8.3 Diffing

```bash
lfv diff docs/note.md
lfv diff docs/note.md snap_01HXYZ
```

### 8.4 Rewinding to an Old Version (auto-branch)

```bash
lfv log docs/note.md
lfv rewind docs/note.md snap_01HXY0
# -> Automatically creates branch rewind/01HXY0/1 and switches the working area to it.
#    Subsequent snaps land on this new branch; the original main branch history is fully preserved.
```

### 8.5 Rename / Move

```bash
# Scenario A: rename via LFV (recommended, atomic)
lfv mv docs/note.md docs/notes/2026-05/note.md
# -> Appends a rename snapshot; path changes from old to new, object unchanged.

# Scenario B: renamed by OS, content unchanged — auto-detection handles it
mv docs/note.md docs/notes/2026-05/note.md
lfv status
#   R   f_01HA7BCD   docs/note.md -> docs/notes/2026-05/note.md
#                    auto-detected (identical content hash)
lfv snap                        # Commit all auto-detected changes

# Scenario C: renamed by OS and content also changed — auto-detection fails, manually declare history continuation
mv docs/note.md docs/notes/2026-05/note-v2.md
$EDITOR docs/notes/2026-05/note-v2.md
lfv status
#   D   f_01HA7BCD   docs/note.md
#                    file missing on disk; possibly moved
#   M   f_01HA7EEE   docs/notes/2026-05/note-v2.md  [newly tracked]
lfv relink f_01HA7EEE --onto f_01HA7BCD
# -> f_01HA7EEE's content (current path + object) is appended as the next Snapshot on f_01HA7BCD's history
# -> Event type derived by comparing f_01HA7BCD's last snapshot with the new Snapshot (typically R+M)
# -> f_01HA7EEE is retired (file_states cancelled, snapshots.log preserved)
```

### 8.6 Delete and Revive

```bash
lfv delete docs/old-note.md     # Mark modified and remove from disk; snap appends deletion marker
lfv snap -m "remove old note"   # Commit deletion marker (or leave it for the next no-arg `lfv snap`)
lfv log docs/old-note.md        # History is still fully queryable
lfv revive docs/old-note.md     # Automatically creates a revive/<...> branch and restores content to working tree
```

### 8.7 Continuing Old History at the Same Path

```bash
# Scenario: docs/old-note.md was previously deleted (has a deletion marker);
#           a new file has been created at the same path.

lfv status
#   M   f_01HA7EEE...   docs/old-note.md  [newly tracked]
#                       note: this path previously existed as f_01HA7BCD (deleted)
#                       to continue its history instead, run:
#                         lfv relink f_01HA7EEE --onto f_01HA7BCD

# Option A: treat the new file as independent, unrelated to old history — snap directly
lfv snap docs/old-note.md -m "new document"

# Option B: the new file is a continuation of the old one — resume old history
lfv relink f_01HA7EEE --onto f_01HA7BCD
# -> f_01HA7EEE's content is appended as the next Snapshot on f_01HA7BCD's history (revive event)
# -> f_01HA7EEE is retired (file_states cancelled, snapshots.log preserved, no active trace)
```

### 8.8 Cross-device Sync

Simply copy or sync the entire working directory (including `.lfv`) to another device via NAS or cloud drive. LFV itself **does not resolve concurrent write conflicts** — the sync tool is responsible for ensuring `.lfv` is not modified concurrently on multiple devices.

## 9. Technology Choices

- **Language**: Rust 2024 edition.
- **CLI framework**: `clap` v4, using the derive-style command tree definition.
- **Error handling**: `thiserror` (library-level error type definitions) + `anyhow` (top-level CLI error aggregation).
- **Hashing**: `blake3` (fast and strong enough).
- **Compression**: `zstd` level 3; defaults `min_bytes` 4 KiB, `max_bytes` 16 MiB, `reject_if_larger: true` (see §5.1).
- **Metadata serialization**: `serde` + strict YAML (human-readable config/metadata) + `serde_json` (JSON Lines row format for `snapshots.log`). All config and metadata files under `.lfv/` use the `.yaml` extension. "Strict YAML" means LFV only writes and accepts a restricted subset: mappings, sequences, strings, numbers, booleans, and null — no anchors, aliases, complex tags, or implicit type coercion.
- **Index storage**: `rusqlite` (embedded SQLite, single-file `index.db`).
- **Text diff**: `similar` (line/word-level diff, unified format output).
- **Time**: `time` or `jiff` (TBD; leaning toward `jiff` for its more modern API).
- **Logging**: `tracing` + `tracing-subscriber`; CLI verbosity controlled by `-v/-vv`.
- **Testing**: `assert_cmd` + `predicates` + `tempfile` for integration tests; unit tests co-located in `mod` blocks.

> Selection principle: prefer libraries broadly validated by the Rust ecosystem; avoid unmaintained crates. All dependencies reviewed quarterly until v1.0.

## 10. Engineering Overview

Module layout:

```
src/
├── main.rs                # CLI entry point: argument parsing and dispatch only
├── cli/                   # clap command definitions and subcommand entry points
│   ├── mod.rs
│   ├── init.rs
│   ├── track.rs
│   ├── snap.rs
│   ├── log.rs
│   ├── diff.rs
│   ├── rewind.rs
│   └── ...
├── repo/                  # Repository abstraction: open, close, path resolution, config
│   ├── mod.rs
│   ├── config.rs
│   └── layout.rs
├── index/                 # SQLite index access layer
│   └── mod.rs
├── object/                # Object store: write, read, compress, GC
│   └── mod.rs
├── snapshot/              # Snapshot entity, serialization, append-write to snapshots.log
│   └── mod.rs
├── branch/                # Branch management
│   └── mod.rs
├── tag/                   # Tag management
│   └── mod.rs
├── diff/                  # Diff algorithm wrapper
│   └── mod.rs
└── util/                  # General utilities: paths, time, strings
    └── mod.rs
```

Test directory:

```
tests/
├── cli_init.rs
├── cli_track.rs
├── cli_snap.rs
├── cli_rewind.rs
└── ...
```

Build and release:

- **Build**: `cargo build` / `cargo build --release`.
- **Test**: `cargo test` (unit tests and integration tests).
- **Formatting**: `cargo fmt`; CI checks with `cargo fmt -- --check`.
- **Static analysis**: `cargo clippy --all-targets -- -D warnings`.
- **Target platforms**:
  - Windows x86_64 (primary development platform);
  - Linux x86_64 (GNU and musl release artifacts);
  - macOS (aarch64 / x86_64) best-effort, not primary test target.
- **Release artifact**: single executable `lfv(.exe)` distributed via GitHub Releases.
- **Versioning**: SemVer. All public command semantics may change before v1.0, but breaking changes must be declared explicitly in the CHANGELOG.

## 11. Roadmap (rough)

- **v0.1**: `init` / `track` / `snap` / `log` / `status` / `show`.
- **v0.2**: `diff` / `rewind` / `branches` / `switch`.
- **v0.3**: `tag` family, `export`, `gc`, `verify`.
- **v0.4**: performance improvements (large files, batch operations), improved error messages.
- **v1.0**: stable CLI semantics, complete documentation, cross-platform CI passing.
