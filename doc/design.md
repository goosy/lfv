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
| File Status | Working-directory files have three statuses: `unmodified`, `modified` (new and deleted files are also `modified`), and `untracked`. Files that are untracked or ignored appear as `untracked` in the status table; that table can be reconstructed from `config.yaml` / `.lfvignore`. |
| Object | A content-addressed storage unit for file content; deduplicated by content hash — different files may share the same object. |
| Snapshot | An "event record" for a tracked file at a point in time: content pointer + path + metadata (message, timestamp, author, parent snapshot). A single Snapshot covers content changes, renames/moves, and deletions. After a file is successfully snapshotted, its status in the status table becomes `unmodified`. |
| Branch | A chain of snapshots for a tracked file; the default branch is `main`. Branch namespaces are independent per file. |
| HEAD | The current branch and latest snapshot pointer for a tracked file. |
| Tag | A human-readable name for a snapshot (optional), used to stably reference a specific version. |
| Action | Actions change the state of a file. Available actions include `track`, `snap`, and `untrack`, plus two actions with no corresponding command: `modify` (achieved by the user editing the file) and `auto-track` / `auto-delete` (applied automatically by LFV during scanning in response to OS file create/delete events; see §6). |

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

- **file_states** (status table): core fields include `fullpath` (repo-relative path), `status` (`untracked` / `modified` / `unmodified`), and `file_id` (null for untracked files). The status table also serves as the "current path index": for tracked, non-deleted files, `fullpath → file_id` is how the CLI resolves paths to files.
- **branches**: per-file branch pointer table.
- **tags**: per-file tag table.
- **head**: per-file current branch and latest snapshot pointer.

## 5. Repository Layout

```
A/                                   # Working directory
├── docs/note.md                     # Example tracked file
├── photos/2025/sunset.jpg
└── .lfv/
    ├── config.yaml                  # Repository-level config (strict YAML)
    ├── HEAD                         # Global placeholder (reserved, mainly for compatibility)
    ├── index.db                     # Status, branches, tags, head index (SQLite or sled)
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

LFV adopts a **track-by-default** policy: files in the working directory should, under normal circumstances, all be in a tracked state. To this end, LFV performs a lazy scan of the working tree on every command invocation (especially `lfv status`, `lfv track`, and `lfv snap`), **automatically responding to OS-level create and delete events** — no background daemon required. It primarily compares the working tree against the status table to detect newly created and deleted files, then updates the status table accordingly.

### 6.2 Auto-track (New Files)

When a scan finds a path in the working tree that is not yet in the status table (and is not excluded by `.lfvignore` or `config.yaml`), it is treated as an "OS-created file." LFV automatically applies the `track` action: assigns a `file-id`, adds it to the tracking list, but **does not immediately append a Snapshot**. At this point the tracking record has no content (content is `null`), which is inconsistent with the actual file on disk, so the status is `modified`. A Snapshot is produced by a subsequent `lfv snap` (with no arguments, it batch-processes all `modified` files).

Auto-track is triggered by: `lfv status` (applied immediately during scan), `lfv track` (with no arguments), and `lfv snap` (before batch-snapshotting with no arguments).

**Cancelling tracking**: if you do not want a file to be auto-tracked, run `lfv untrack <file>`. LFV writes the path into the `untracked` list in `config.yaml` and updates the status table to reflect this. For files that already have history, they will no longer be tracked and will produce no new history. For files that were auto-tracked, since no Snapshot exists at that point, no deletion-marker snapshot will be produced and no trace will be left in the repository.

For files marked via `lfv untrack <file>`, because the status table has the cached state and `config.yaml` has them in the `untracked` list, subsequent scans will not auto-track them again.

**Friendly hint for same-path new files**: if a path previously had a deleted `file-id` (i.e. a deletion marker was appended for that path), and auto-track assigns a new `file-id` at the same path, `lfv status` will append a hint below that file's entry:

```
  M   f_01HA7EEE...   docs/old-note.md  [newly tracked]
                      note: this path previously existed as f_01HA7BCD (deleted)
                      to continue its history instead, run:
                        lfv mv f_01HA7BCD f_01HA7EEE
```

This hint appears only when auto-track has assigned a new `file-id`; once the new `file-id` records its first Snapshot, it stands as an independent file and the hint disappears.

For `untracked` files that still have history (no deletion marker), running `lfv track <file>` again first removes the entry from the dynamic `untracked` list in `config.yaml`, then resumes tracking under the original `file-id` and updates the status table to `tracked`. Note that this differs from tracking a *deleted* file (which has a deletion marker), where a new `file-id` is always assigned by default unless the user explicitly runs `lfv revive`.

### 6.3 Auto-delete (OS-deleted Files)

When a scan finds that a **tracked file's** path has disappeared from the working tree, and there is no evidence of a rename (i.e. rename auto-detection could not pair it), the file is treated as "OS-deleted."

LFV does **not** immediately append a deletion-marker Snapshot; instead it marks the file `D` (suspected delete). When `lfv snap` is subsequently run with no arguments, all files in `D` status have their deletion automatically confirmed — a Snapshot with `object=null` is appended and the corresponding `file-id` is removed from the current-path index in the status table.

This design gives the user a window before `lfv snap`: while a file is still in `D` status, they can still run `lfv mv f_old <new-path>` to reclassify it as a rename, avoiding accidental deletion.

### 6.4 Rename / Move (`lfv mv`)

- `lfv mv <src> <dst>`: migrates the tracked file corresponding to `src` to the `dst` path and appends a Snapshot. Both `<src>` and `<dst>` may be a path or a `file-id` (`f_*` prefix).
  - If `src` still exists in the working tree and `dst` does not, the CLI first moves the file to `dst` on disk, then appends the snapshot (atomic semantics).
  - If `src` no longer exists (already moved by the OS or editor) and `dst` already contains the file, `lfv mv` only "records the snapshot" without touching the disk.
  - The new Snapshot's `object` is determined by the hash of `dst`'s current content: if it matches the parent snapshot's object, the event renders as pure `R`; otherwise as `R+M`.
  - The default `message` is `rename: <old-path> -> <new-path>`.
- **Auto-detection vs. manual registration**:
  - Auto-detection applies only when content hashes are **exactly equal** (`rename.autodetect`, enabled by default). When a match is found, `lfv status` displays an `R` line and automatically pairs the files.
  - If the OS moves a file **and** its content changes, auto-detection fails — `lfv status` will display both a `D` line (old `file-id` missing from its original path) and a `?` line (new file at the new path, with a `u_*` handle). The user reviews and explicitly registers the rename with `lfv mv f_old u_new`. This is the standard channel for merging an "OS move + content edit" into a single Snapshot chain event.
  - Auto-detection can also be disabled (useful for bulk rename + edit scenarios to avoid mispairing).
- History display: `lfv log <file>` renders rename events as "`R` old-path -> new-path", alongside `A`(add) / `M`(modify) / `D`(delete) / `R+M`(rename+modify) (see §5.2.1).

### 6.5 `lfv delete`

`lfv delete <file>`: marks the file as `modified` in the status table and deletes it from disk. On the next `lfv snap`, a Snapshot is appended for the corresponding `file-id` — `path = <current path>`, **`object = null`** (deletion marker). After this, the `file-id` can no longer be resolved via the current path, but the last known path is preserved in the deletion-marker snapshot; `lfv list --deleted` can still display it by path.

**Complete history is preserved**: the file can still be queried via `lfv log` / `lfv show` / `lfv diff`; to "revive" it, use `lfv revive` (see §7.2) or `lfv rewind <snap>` — both automatically create a new branch (without disturbing the existing deletion event).

### 6.6 Exclusion Mechanisms: `.lfvignore` and `config.yaml`

Track-by-default would otherwise include temporary files, build artifacts, etc., so an ignore mechanism is necessary:

- The `.lfvignore` file at the working directory root (syntax compatible with `.gitignore`) lists **statically excluded** path patterns that must never be tracked;
- `config.yaml` stores **dynamic tracking configuration** maintained by LFV commands — for example, per-path `untracked` entries written by `lfv untrack <file>` and removed by `lfv track <file>`;
- `.lfvignore` takes priority over `config.yaml`: if a path matches `.lfvignore`, `lfv track <file>` must error and prompt the user to edit `.lfvignore`; dynamic config cannot override a static exclusion;
- The `.lfv/` directory itself is always implicitly excluded; no `.lfvignore` entry is needed.

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
| `lfv untrack <file>` | Stop tracking this path: write it into the dynamic `untracked` list in `config.yaml` and update the status table entry to `untracked`. Full history is preserved; the file can be re-tracked at any time via `lfv track <file>`, and then snapshotted or revived. |
| `lfv mv <old> <new>` | Record a rename pre-mark in the status table for the `file-id` of `<old>`, targeting `<new>`. Whether an actual file move is performed in the working tree is governed by §6.4. |
| `lfv delete <file>` | Mark the file as `modified` in the status table and delete it from disk. On the next `lfv snap`, when LFV detects it as `modified` with no file present, it appends a `object=null` deletion-marker snapshot and removes the entry from the `untracked` list in `config.yaml`. Updates the cached status in the status table. Full history is preserved and the file can be revived at any time. |
| `lfv revive <ref>` | Revive a deleted file. `<ref>` may be a `file-id`, the last known path, or a specific Snapshot id. Automatically creates a new branch (`revive/<...>`) and restores the content to the working tree from the selected snapshot. |
| `lfv list [--deleted]` | List all tracked files with their current branch and latest snapshot summary. By default shows only active files; `--deleted` also lists files with a deletion marker. |

### 7.3 Status and Snapshots

| Command | Description |
| --- | --- |
| `lfv status [<file>]` | **Without `<file>`: list all `modified` tracked files in the repository.** Runs a lazy scan first (auto-tracks newly created files, skips `untracked` entries in the status table). Normal output shows only currently tracked files, each line including the `file-id` (`f_*`). With `<file>`: show the status of that file only. |
| `lfv snap [<file>] [-m <msg>]` | Create a new snapshot for a file. **Without `<file>`: batch-snapshot all `modified` tracked files.** Refuses if working content is unchanged (unless `--allow-empty`). |
| `lfv log <file>` | List the snapshot history for a file. Supports `--branch <name>`, `--graph`, `--limit N`. |
| `lfv show <file> <snap>` | Output metadata for a specific snapshot; `--content` outputs the content; `--out <path>` exports it. |

Notes:
- For `modified` files in `lfv status`: deleted files show as `D`; files whose path differs from the last snapshot show as `R`; files with no snapshot yet show as `A`; all others show as `M`; files with both content and path changes show as `R+M`.
- During `lfv snap`: `D` files get a deletion marker appended; other modified files get a content snapshot; `unmodified` files are skipped. A lazy scan runs first (auto-tracking new files).
- To also show untracked files (statically excluded by `.lfvignore` or dynamically excluded by `config.yaml`) in `lfv status`, use `--include-untracked`. They are listed separately at the bottom as `~` lines, without a `file-id` column (since they have none).

#### 7.3.1 `lfv status` Output Format

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
                        lfv mv f_01HA7BCD f_01HA7EEE

Ignored / excluded (not auto-tracked):
  ~   docs/scratch/tmp.bin             (843 KB, matched .lfvignore)
  ~   drafts/local.md                  (2.1 KB, untracked by config.yaml)
```

**Status flags:**

- `A` = add (newly tracked, no snapshot yet)
- `M` = modified (includes newly tracked files not yet snapshotted)
- `R` = rename (auto-detected or registered via `lfv mv`; no distinction needed)
- `D` = suspected delete (missing from disk; will be confirmed as a deletion marker on the next `lfv snap`)
- `~` = not tracked (excluded by `.lfvignore` or `config.yaml`; not auto-tracked)

**The `file-id` column:**

Displays `f_*` (ULID with `f_` prefix), abbreviated to the first 10 characters by default; `--long` shows the full length. Assigned at track time and persists through the file's entire lifecycle — it is the only stable reference token at the CLI level.

**Any command that accepts `<file>` also accepts either a path or `f_*` as its argument:**

- `lfv mv f_01HA7BCD f_01HA7EEE` — the old `file-id` takes over the path of the new `file-id`, appending an `R+M` snapshot; the new `file-id`'s tracking record is cancelled.
- `lfv mv docs/old-note.md docs/notes/new.md` — equivalent form using paths.
- `lfv delete f_01HA7DEF` — can explicitly register a deletion by `file-id` even when the file is no longer in the working tree.

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
# -> Lazy scan: all files in the working tree not excluded by .lfvignore or config.yaml
#    are auto-tracked (tracking flag set, no Snapshot yet).
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

# Scenario C: renamed by OS and content also changed — auto-detection fails, manual registration needed
mv docs/note.md docs/notes/2026-05/note-v2.md
$EDITOR docs/notes/2026-05/note-v2.md
lfv status
#   D   f_01HA7BCD   docs/note.md
#                    file missing on disk; possibly moved
#   M   f_01HA7EEE   docs/notes/2026-05/note-v2.md  [newly tracked]
lfv mv f_01HA7BCD f_01HA7EEE    # Old file-id takes over new path; new file-id cancelled.
# -> Appends an R+M snapshot (both path and object changed)
```

### 8.6 Delete and Revive

```bash
lfv delete docs/old-note.md     # Remove from status table's current path index; snap appends deletion marker
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
#                         lfv mv f_01HA7BCD f_01HA7EEE

# Option A: treat the new file as independent, unrelated to old history — snap directly
lfv snap docs/old-note.md -m "new document"

# Option B: the new file is a continuation of the old one — resume old history
lfv mv f_01HA7BCD f_01HA7EEE
# -> f_01HA7BCD takes over docs/old-note.md and appends a revive snapshot
# -> f_01HA7EEE's tracking record is cancelled (no snapshot, no trace)
lfv snap docs/old-note.md -m "resumed from old history"
```

### 8.8 Cross-device Sync

Simply copy or sync the entire working directory (including `.lfv`) to another device via NAS or cloud drive. LFV itself **does not resolve concurrent write conflicts** — the sync tool is responsible for ensuring `.lfv` is not modified concurrently on multiple devices.

## 9. Key Design Decisions

1. **Single-file scope**: all operations must explicitly specify `<file>`; there is no "global snapshot." This follows directly from the project's founding motivation.
2. **Rewind never destroys history**: any "go back in time" operation is implemented via **creating a new branch**, ensuring no snapshot before HEAD ever becomes unreachable.
3. **Append-only history**: `snapshots.log` is never rewritten, facilitating backup, auditing, and crash recovery.
4. **Content addressing + deduplication**: even with massive duplicate content across thousands of independent files, the object store holds only one copy.
5. **SQLite as the index**: metadata requiring random updates — status table, HEAD, branches, tags — lives in `index.db`; purely historical data lives in files.
6. **Small-tool philosophy**: CLI subcommands are clear, composable, and scriptable; no premature abstraction toward a GUI or service.
7. **Only two storage object types**: the only truly immutable storage objects in the repository are Object and Snapshot; no tree. Everything else is mutable index (see §4.2).
8. **Snapshot id separated from tamper-resistance**: Snapshot ids use ULID for human readability and time ordering; tamper-resistance is handled by the dedicated `digest` field, verified by `lfv verify` (see §5.2).
9. **Path is a field on a Snapshot, not the file's identity**: `file-id` is fully decoupled from path; rename/move is recorded as an **immutable event**, structurally identical to "content change." This makes the Snapshot chain the complete source of truth for both file location and content, eliminating truth-source fragmentation from a mutable `aliases` list.
10. **Deletion = deletion marker; history is never lost**: `lfv delete` removes the file from disk and marks its status-table entry as `modified`. `lfv snap` then confirms the deletion by appending an `object=null` Snapshot and removing the file record from the status table. All historical snapshots are fully preserved and the file can be revived at any time via `lfv revive`.
11. **`file-id` is the only stable reference**: all commands that accept `<file>` also accept a path or `f_*`. A `file-id` is assigned at track time and persists through the file's entire lifecycle — including after deletion — and is the only still-valid reference token after the file disappears or its path changes.
12. **Track-by-default policy**: files in the working directory are, under normal circumstances, always in a tracked state. LFV performs a lazy scan on every command invocation: newly created files are auto-tracked; tracked files deleted by the OS are automatically flagged `D` and receive a deletion marker on the next `lfv snap`. This matches the mental model of "file-centric version management" (see §6).
13. **Snapshot topology is a directed tree, not a DAG**: each Snapshot has exactly one parent pointer, forming a directed forest. Branches diverge and evolve independently; there is no topological merge point. To "realign" two branches, the user must explicitly run `lfv rebase` (not implemented yet) — branches never merge automatically. At the UI layer, file-hash (the object's blake3) serves as the measure of content identity, so `lfv log --graph` and `lfv branches` can show when two branches share the same content at a given point, while each Snapshot's identity (ULID) remains unique and independent. In LFV's single-user, single-file, local scenario this design incurs almost no cost while significantly reducing the implementation complexity of the storage layer, index layer, and `log` rendering.
14. **Dual compression thresholds**: `min_bytes` (floor, default 4 KiB) and `max_bytes` (ceiling, default 16 MiB) handle files that are too small or too large to compress; `blake3` is always computed over raw bytes; objects that skip compression or are rejected as ineffective are stored as `.raw`, compressed objects as `.zstd` (see §5.1).

## 10. Technology Choices

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

## 11. Module Layout (initial)

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

## 12. Build and Release

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

## 13. Roadmap (rough)

- **v0.1**: `init` / `track` / `snap` / `log` / `status` / `show`.
- **v0.2**: `diff` / `rewind` / `branches` / `switch`.
- **v0.3**: `tag` family, `export`, `gc`, `verify`.
- **v0.4**: performance improvements (large files, batch operations), improved error messages.
- **v1.0**: stable CLI semantics, complete documentation, cross-platform CI passing.

## 14. Possible Future Extensions (non-committal)

- Simple HTML5 interface with visual branch-tree display for a file.
- Hook system, e.g. auto-snap before saving.
- Cross-repository object pool sharing.
- Bridges to git LFS and NAS vendor APIs.

## 15. Open Questions

The following will be resolved based on practical feedback during development:

1. Should `index.db` ultimately use `rusqlite` or `sled` / `redb`?
2. Does binary file diffing need a friendlier extended mode such as "image thumbnail diff"?
3. Should `lfv status --include-untracked` recursively scan the entire working tree to discover files statically excluded by `.lfvignore` and dynamically excluded by `config.yaml`? Performance vs. usability trade-off.
4. Threshold for rename auto-detection (`rename.autodetect`): detect only on exact content-hash match, or allow approximate matching at "similarity ≥ N%"? The latter is significantly more complex; leaning toward exact match first.
5. For `lfv revive`, which snapshot should content be restored from by default — the last non-deletion-marker snapshot before deletion, or should the user be required to specify `<snap>` explicitly? Leaning toward the former as default, with user override available.
6. For a file at a path that was soft-deleted and then `lfv track`ed again: should the old `file-id` be reused (automatically continuing history) or a new `file-id` assigned (treating it as a different file)? Leaning toward **assigning a new `file-id`** — a same-name file reappearing is not necessarily semantically the same file; auto-continuing history risks being misleading. Users who want to resume can explicitly run `lfv revive`.
