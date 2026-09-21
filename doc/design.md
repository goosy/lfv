# LFV Implementation Design

> This document is the implementation design for `spec.md` (hereafter spec): it answers "how to build it" — storage structure, internal mechanisms, engineering structure. Behavior and constraints are governed by the spec. For the Chinese version, see `design.zh-cn.md`.
> References of the form `spec §x` point to sections of the specification.

## 0. Overview

In one sentence: **the source of truth is the filesystem (`objects/` + `snapshots.log` + the yaml files), `index.db` is only a cache; every command = lazy scan → resolve arguments → one ops flow → render.**

Data falls into four layers, with dependencies running top to bottom:

| Layer | Contents | Mutability | Module |
| --- | --- | --- | --- |
| Object | File Object / Tree Object | written once, never changed | `object` |
| Snapshot | File Snapshot / Tree Snapshot (JSON Lines) | append-only | `snapshot` |
| Refs (user intent) | `HEAD`, `branches.yaml`, `tags.yaml`, `trees/HEAD`, `trees/tags.yaml`, `config.yaml`, `meta.yaml`, `REPLAY.yaml` | mutable, not rebuildable | `reftable`, `tracked`, `tree`, `repo` |
| Index | `index.db` | mutable, rebuildable | `index` |

## 1. Engineering Structure

### 1.1 Crate layout: a single crate, lib + bin

- One package `lfv`; `src/lib.rs` exports all core modules, and `src/main.rs` only does clap parsing + dispatch + error output/exit codes.
- **No workspace** (no `lfv-core` / `lfv-cli` split). Rationale: in v0.x there is a single author and the only consumer is the CLI; splitting into a workspace would only add version and path maintenance cost. The lib/bin separation is already enough for `tests/` to run end-to-end through `assert_cmd` and also call the ops layer directly for fast tests that start no process.
- If a GUI or daemon layer needs to reuse the code later, `lib.rs` is already there and upgrading to a workspace is a mechanical change.

### 1.2 Module list and responsibilities

```
src/
├── main.rs        entry: parse -> dispatch -> error formatting/exit code
├── lib.rs         pub mod declarations; no logic
├── cli/           clap derive command tree; one file per subcommand; only "arguments -> ops call -> render"
├── ops/           use-case layer: one function per flow in §4; the only initiator of cross-module transactions
├── repo/          Repository: find the repo root, .lfv layout paths (layout.rs), config.yaml (config.rs), process lock, init/open
├── object/        ObjectStore: hash, compression policy, bucketed write/read, enumerate/remove (gc); TreeManifest (Tree Object codec)
├── snapshot/      Snapshot records, canonical digest form, SnapshotLog (append / full read / tail truncation), event-type derivation, ancestor traversal
├── reftable/      name -> snap_id YAML tables (branches.yaml / tags.yaml / trees/tags.yaml share one format)
├── tracked/       handle for the files/<ULID>/ directory: meta.yaml, HEAD, branch table, tag table, log; file-id allocation and retirement
├── tree/          handle for the trees/ directory: log, HEAD, tree: tags; Tree Object construction
├── index/         SQLite: schema, migration, queries, rebuild
├── scan/          working-tree scan: .lfvignore, incremental invalidation, auto-track / auto-delete, automatic rename detection, status-flag derivation; working-area access layer (Unicode normalization, locating, case-collision detection)
├── resolve/       CLI argument resolution: <file> <FO-ish> <FS-ish> <TO-ish> <TS-ish> -> strongly typed targets
├── diff/          unified diff of two byte strings (text) / metadata diff (binary)
├── merge/         three-way text merge, conflict markers, replay engine, in-progress state (merge/rebase --continue/--abort)
└── util/          path normalization, time, ULID, atomic write (tmp + rename), small filesystem helpers
```

Branches and tags have no module of their own: both use the same `name: snap:<ULID>` YAML format and differ only in behavior ("a tag is only added, never repointed"), which one generic table in `reftable` plus each plane's handle covers.

### 1.3 Dependency direction (downward only)

```
cli → ops → { scan, resolve, merge, diff }
                 → { tracked, tree, index, object, snapshot, reftable }
                        → repo → util
```

- `ops` is the only layer allowed to touch `tracked`, `tree`, `index`, and `object` at the same time and to decide the write order (§3.9 crash consistency depends on this).
- `scan` writes only `index` and `tracked` (auto-track has to allocate a file-id and create its directory); it never writes snapshots.
- `resolve` is read-only.
- `cli` must not call `index` / `object` directly; all output data comes from structs returned by ops (which leaves room for a future `--json` and makes test assertions easier).

### 1.4 Third-party crates (cf. spec §6 technology choices)

| Purpose | Crate | Notes |
| --- | --- | --- |
| CLI | `clap` v4 derive | — |
| Errors | `thiserror` + `anyhow` | `thiserror` in the library layers, `anyhow` at `ops` and above |
| hash / compression | `blake3` / `zstd` | — |
| Serialization | `serde` + `serde_json` + `serde_yaml_ng` | `serde_yaml` is archived; only the mapping/sequence/scalar subset is used |
| SQLite | `rusqlite` (`bundled`) | no system dependency, Windows being the primary platform |
| diff / three-way merge | `diffy` | conflict-marker labels are substituted at line start on the output by the `merge` module (spec §4.7.3) |
| Time | `jiff` | — |
| ID | `ulid` | — |
| ignore rules | `ignore` (ripgrep's gitignore implementation) | see §4.2 |
| Unicode normalization | `unicode-normalization` | working-area access layer only, see §4.2.1 |
| Logging | `tracing` + `tracing-subscriber` | — |
| Testing | `assert_cmd` + `predicates` + `tempfile` | — |

### 1.5 Test directory

```
tests/
├── cli_init.rs
├── cli_track.rs
├── cli_snap.rs
├── cli_rewind.rs
└── ...
```

- Unit tests live next to the code in each module's `#[cfg(test)]`; `snapshot::event_kind` and `snapshot::loop_check` must cover every combination in the §3.3.1 table.
- Integration tests invoke the executable through `assert_cmd`; when internal state has to be inspected, they open `.lfv` with the library directly.

### 1.6 Build and release

- **Build**: `cargo build` / `cargo build --release`.
- **Test**: `cargo test` (unit tests and integration tests).
- **Formatting**: `cargo fmt`, with `cargo fmt -- --check` in CI.
- **Static checks**: `cargo clippy --all-targets -- -D warnings`.
- **Target platforms**:
  - Windows x86_64 (primary development platform);
  - Linux x86_64 (two release artifacts, GNU and musl);
  - macOS (aarch64 / x86_64) supported on a best-effort basis, not primarily tested.
- **Release artifact**: a single executable `lfv(.exe)`, distributed through GitHub Releases.
- **Version numbers**: SemVer; before 1.0 the semantics of any public command may be adjusted, but breaking changes must be stated explicitly in the CHANGELOG.

## 2. Core Data Model

### 2.1 Identifiers (all newtypes; a bare String must never cross a module boundary)

| Type | Text form | Notes |
| --- | --- | --- |
| `FileId` | `file:<ULID 26>` | the text form (DB, YAML, logs, CLI) carries the `file:` prefix; directory names under `files/` are the bare `<ULID>` (Windows file names cannot contain `:`) |
| `SnapId` | `snap:<ULID 26>` | the same format on both the file and tree planes, globally unique |
| `ObjectHash` | `blake3:<64 hex>` | `[u8; 32]` internally; File Object and Tree Object share one type, and **the storage layer does not distinguish them** |
| `TreeId` | = `ObjectHash` | a content hash, with no identity of its own |
| `BranchName` / `TagName` | full naming rules in spec §4.6 (no `:` or control characters, no YAML indicator characters or leading/trailing whitespace, non-empty, does not start or end with `/`, not a namespace keyword, and no collision between a branch name and a tag name of the same file); §2.1.1 of this document explains where those constraints come from | tree tags are stored internally as `tree:<name>`, with `TreeTag` as a separate type |
| `RepoPath` | relative to the repo root after normalization (CLI input may be relative to cwd, or start with `/` for a repository-absolute path), `/`-separated, UTF-8, following the Windows file-name rules (no `<>:"\|?*` or control characters, no component ending in `.` or a space, no reserved names) | spec §4.2 path rules |
| `WorktreeRef` | `work:<path>` | the literal for the special working-area File Object |

### 2.1.1 Serialization rules for metadata strings (corresponds to spec §3.5)

Strings written into these YAML files fall into three categories by how controllable they are, handled differently:

**User-controlled** (`RepoPath`, `config.yaml`'s `user.name`, a snapshot's `author`) — always double-quoted on serialization, escaped inside the quotes by the standard YAML double-quoted-string rules, reusing the same escaping implementation as digest canonicalization (§3.3.2) (only `"`, `\`, and control characters are escaped; non-ASCII is raw UTF-8). `RepoPath` already forbids `"` and `\` (spec §4.2), so this rule needs no real escaping for it in practice — a scan to the next `"` is the whole value; `user.name`/`author` have no character restriction and do need full escaping under this rule.

**User-proposed, LFV-approved** (`BranchName`, `TagName`) — not quoted on serialization. The full naming rules are laid down in spec §4.6; this section explains why they look the way they do: the price of leaving the value unquoted is that creation-time validation must reject anything that would create ambiguity in a bare YAML scalar.

| Rule | Why |
| --- | --- |
| no control characters and no `:` | control characters break line-oriented files and output; `:` is the namespace separator |
| no `#?,[]{}&*!\|>'"%@` or the backtick itself | YAML indicator characters (c-indicator), which carry special syntactic meaning in specific positions of a bare scalar |
| no `\` | keeps the value escape-free should it ever need to appear somewhere that requires escaping |
| no leading or trailing whitespace | a bare YAML scalar has its surrounding whitespace trimmed, so the value read back would differ from what was written — silent corruption, not a parse error |
| the whole name not equal to `-`, and not matching `/^-\s/` (a hyphen immediately followed by whitespace) | a leading `-` plus whitespace is YAML's block-sequence-entry marker; `-` elsewhere (not followed by a space, at the end, or in the middle) is fine |

The check belongs in the commands that create branch/tag names (`lfv branch-rename`, `lfv tag`, and the preserved branch names and `tree:detour/...` tags implicitly created by rewind/detour/rebase/revive), at the same layer as the namespace-keyword check and the branch/tag collision check, refusing creation outright on a violation.

**LFV-controlled** (`snap:<ULID>`, `file:<ULID>`, `blake3:<hex>`, RFC 3339 timestamps, `true`/`false`, integer version numbers, and the like) — the character set is defined by LFV itself and known safe; written bare, unquoted and unescaped.

### 2.2 The object plane

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

The write policy is in §3.2.

**`object::TreeManifest`** (the in-memory form of a Tree Object; format in §3.5)

```rust
pub struct TreeManifest(BTreeMap<RepoPath, ObjectHash>);   // BTreeMap order == UTF-8 byte order
impl TreeManifest {
    pub fn to_bytes(&self) -> Vec<u8>;          // `- "path": blake3:...\n` per entry; empty => zero bytes
    pub fn parse(b: &[u8]) -> Result<Self>;     // line regex `^- "(.+)": (\S+)$`
    pub fn tree_id(&self) -> ObjectHash;        // blake3(to_bytes())
    pub fn diff(&self, other: &Self) -> Vec<TreeEntryChange>;
}
```

### 2.3 Snapshot records

A single wire structure shared by both planes; plane-specific invariants are checked in the constructor:

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

- There is no `tags` field: the source of truth for tags is `tags.yaml` and tags can be deleted, whereas the record is immutable and protected by its digest — the two cannot coexist.
- The canonical digest form is in §3.3.2; event-type derivation `snapshot::event_kind(parent: Option<&Snapshot>, cur: &Snapshot) -> EventKind` is in §3.3.1.

### 2.4 `snapshot::SnapshotLog` (append-only JSON Lines)

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

- Each file-id's log is read into memory as a whole (a single file's history is usually tens to a few thousand lines); the same applies to the tree log. **Snapshot bodies are not cached in `index.db`** — only locator information is (§3.6, `snap_locator`).
- Torn tail: if the last line has no `\n` or fails to parse as JSON, it is treated as an unfinished write and ignored on load; before an `append`, if the last byte is not `\n`, that partial line is truncated first. **A complete line whose digest does not match is not handled automatically** — it is left for `verify` to report.
- `parent` must already exist in the same log (checked on append).

### 2.5 `tracked`: the file-id directory

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

- `TrackedStore::create(initial_path) -> FileId`: allocates a ULID, creates the directory, writes `meta.yaml`, `HEAD = main`, an empty `branches.yaml` (the branch is written only with the first snapshot, §3.8), and an empty log.
- `head_snapshot()`: `branches[head]` → `log.get`; **the current branch being absent from the branch table** means the file is tracked but has no snapshot yet (status `A`).
- Invariants: every value in `branches` exists in the log; the HEAD branch name is in `branches` unless the log is empty.

### 2.6 `tree`: the tree plane

```rust
pub struct TreePlane { log: SnapshotLog /* plane = Tree */, head: Option<SnapId>, tags: RefTable<TreeTag> }
```

`trees/HEAD` being an empty file ↔ `head = None` (§3.7). There are no branches. Reachability: `trees/HEAD` and all `tree:` tags are roots; when `rewind <TS-ish>` leaves a HEAD that has no tag and is not an ancestor of the target, a `tree:detour/<anchor-short>/<n>` tag is applied automatically (spec §4.5.1).

### 2.7 Working-area state model (shared by `scan` and `index`)

Each tracked row in `index.db` stores only `status ∈ {modified, unmodified}` plus factual fields; **the status flag is derived at render time** and never persisted:

```
fn flag(row, head: Option<&Snapshot>) -> Flag
  head == None                                     -> A        (tracked, no snapshot yet)
  !row.present                                     -> D        (head.object is Some; row kept on purpose)
  row.path != head.path && row.hash == head.object -> R
  row.path != head.path && row.hash != head.object -> R+M
  row.hash != head.object                          -> M
  else                                             -> unmodified
```

`head == None` always renders as `A`, regardless of whether it is a brand-new file or a path that previously had a deleted file-id — the latter only adds a hint line under the `A` row (§4.4) and does not change the flag itself.

`untracked` rows live in a table of their own (§3.6) and record only paths that exist on disk (§4.2).

Invariant: **at any moment a path belongs to at most one active (non-retired, non-untracked) file-id**; a `D` row occupies the path of its HEAD snapshot, and if a file reappears at that path the scan treats it as the same file returning (`M` or unmodified), not as a new file.

### 2.8 Argument resolution (`resolve`)

The argument types from the preamble of spec §4 resolve to:

```rust
pub struct FileRef   { id: FileId }                                                  // <file>
pub struct SnapTarget { file: Option<FileId> /* None = tree plane */, snap: SnapId }  // <FS-ish>
pub enum FileTarget  { Worktree(FileId, RepoPath), Object(FileId, ObjectHash) }      // <FO-ish>
pub enum TreeTarget  { Object(ObjectHash), Snap(SnapId) }                            // <TO-ish> / <TS-ish>
```

Disambiguation order for a bare token (purely syntactic, no index lookup): the `snap:` prefix (snap-id; `snap_locator` tells whether it is on the file or tree plane) → the `file:` prefix (file-id) → the `work:` prefix (working-area literal) → the `tree:` prefix (tree tag) → the `blake3:` prefix (tree-id) → when it contains `:`, it is split at the first `:` as `<ref>:<file>`, and the `<file>` part is classified again by these rules as a file-id or a path → otherwise treated as a path (`a`, `./a`, `../a` relative to cwd, `/a/b` repository-absolute; operating-system absolute paths and drive letters are rejected; `\` in input is read as `/`, and several leading `/` count as one; normalized into a `RepoPath`, an error if it leaves the repository; looked up in `tracked.path`, then in `tracked.head_path` (a rename pending write), then as the last non-retired file-id that held that path in history, taking among several candidates the one whose HEAD snapshot has the latest `created_at` and failing that the largest snap-id). Branch names and tag names may not appear as bare tokens; in commands that already carry a `<file>` argument, a bare token in the `<FS-ish>` position is resolved first as a snap-id, then as a branch name of that file, then as a tag name; a branch and a tag of the same file can never share a name (spec §4.6 blocks that at creation), so the last two steps cannot conflict.

Which file-id / plane a bare `snap:*` belongs to is answered by `index.snap_locator` (§3.6).

The `FileId` carried by `FileTarget::Object` is only the file context the resolution happened to pass through (used, for instance, by `lfv diff <FO-ish>` to infer the implicit second argument `work:<path>`) — it does not mean the File Object belongs to that file. A File Object is a content-addressed storage unit; different file-ids and different branches may reference the same hash (spec §3.2.1). For `lfv show <FO-ish>` (without `--content`/`--out`) to list "every snapshot referencing this hash", it must scan all of `files/*/snapshots.log` and match `object == target hash`, not just the one file-id the resolution happened to carry. This is a repo-wide scan; there is no index for it today — it is a low-frequency diagnostic command, so a full scan is acceptable, with a reverse `object_hash -> snap_id` index left for later if this becomes a bottleneck.

## 3. Storage Structure

### 3.1 Directory layout

```
A/                                   # working directory
├── docs/note.md                     # example of a tracked file
├── photos/2025/sunset.jpg
└── .lfv/
    ├── config.yaml                  # repository-level configuration (strict YAML), including the format version
    ├── lock                         # process-level advisory lock, an empty file
    ├── index.db                     # rebuildable cache of state/branches/tags/tree_file_refs (SQLite)
    ├── objects/                     # content-addressed object store (File Object + Tree Object)
    │   ├── tmp/                     # objects being written (does not collide with two-hex-digit bucket names)
    │   ├── ab/
    │   │   ├── cdef0123...zstd      # zstd-compressed File Object
    │   │   └── cdef0123...raw       # as-is File Object (too small / too large / incompressible)
    │   ├── 7f/
    │   │   └── a3bc9d12...raw       # Tree Object (manifest YAML, usually small -> .raw)
    │   └── ...
    ├── files/                       # metadata for each tracked file
    │   ├── <ULID>/                  # directory name = the file-id without its `file:` prefix
    │   │   ├── meta.yaml            # file-level metadata (creation time, initial path, retirement marker)
    │   │   ├── HEAD                 # current branch name
    │   │   ├── branches.yaml        # this file's branch table
    │   │   ├── tags.yaml            # this file's tag table
    │   │   ├── snapshots.log        # append-only snapshot records (JSON Lines)
    │   │   └── REPLAY.yaml          # in-progress merge/rebase state; exists only during the operation
    │   └── ...
    ├── trees/                       # global metadata
    │   ├── snapshots.log            # append-only global snapshot records (JSON Lines)
    │   ├── tags.yaml                # tree tag table
    │   └── HEAD                     # current tree-head, holding snap:<ULID> or empty
    └── logs/                        # CLI operation logs (optional, useful for debugging; not implemented in v0.x)
```

### 3.2 Object store

- Content addressing: an object's identity = `blake3(raw bytes)`; the hash prefix in the bucket path comes from the raw content and is independent of whether the object is compressed;
- Bucketing: the first 2 hex characters form the directory name, avoiding too many files in one directory;
- Compression: the raw bytes are compressed with `zstd` (level **3** by default) before being written, with the extension `.zstd`; objects are decompressed in memory when read;
- Stored as-is (extension `.raw`): compression is skipped when **any** of the following holds:
  - **floor**: `size < min_bytes` (default **4 KiB**, 4096 bytes) — too small, and framing overhead may make the compressed form larger;
  - **ceiling**: `size >= max_bytes` (default **16 MiB**, 16777216 bytes) — avoids the memory peak of reading a large file in one piece;
  - **ineffective compression**: after an attempt, `compressed_len >= original_len` (`reject_if_larger`, default **true**) — for content such as already-compressed binaries with nothing to gain;
- Sharing across files and branches: identical content is stored only once, saving space.

**Write flow** (`ObjectStore::put_path`):

- `size < min_bytes` or `size >= max_bytes` → single pass: hash and write to `objects/tmp/<random>` while reading, then rename to `objects/ab/<rest>.raw` once the hash is known. Large files never enter memory as a whole.
- Otherwise read into memory, hash, and zstd-compress; if `compressed_len >= original_len` and `reject_if_larger`, store `.raw`, otherwise `.zstd`.
- If the target already exists → simply discard the temporary file (deduplication). Only one extension may exist for a given hash; `verify` reports it when both are present.
- The storage layer does not distinguish File Objects from Tree Objects; a zero-byte file and an empty manifest are the same object, which is expected behavior.

#### 3.2.1 Default compression config (`config.yaml`)

```yaml
compression:
  enabled: true
  algorithm: zstd
  level: 3              # zstd 1..=22
  min_bytes: 4096       # floor: size < min => .raw
  max_bytes: 16777216   # ceiling: size >= max => .raw
  reject_if_larger: true
```

### 3.3 Snapshot record format

Each line of `snapshots.log` is one JSON object (JSON Lines):

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

Field notes:

- `id`: a monotonic, readable snapshot identifier using a [ULID](https://github.com/ulid/spec) with the `snap:` prefix. A ULID carries a timestamp prefix plus a random suffix, which makes chronological sorting in `log` easy and makes the id convenient for a human to paste in a terminal.
- `parent`: the parent snapshot id; `null` for the first snapshot on a branch.
- `path`: **the file's relative path in the working tree at the moment of this snapshot**. A rename/move event shows up exactly as this field differing from `parent.path`; when the file does not currently exist, the last known on-disk path is kept (which keeps `log` readable).
- `object`: the blake3 hash of the referenced Object (with the `blake3:` prefix so the algorithm can be switched later); **`null` when the file does not currently exist**.
- `size`: the byte size of the corresponding Object; `0` when `object = null`.
- `created_at`: UTC, second precision, RFC 3339 with the `Z` suffix.
- `author`: taken from `user.name` in `config.yaml`, defaulting to the operating-system user name.
- `digest`: **the `blake3` hash of the canonical serialization of every field of this snapshot record except `digest`**, used for tamper detection; the canonical form is in §3.3.2. `lfv verify` recomputes and compares it line by line.
- The record **contains no tag**: a tag is an external mutable pointer (§3.8).

A snapshot does not record the branch it belongs to — a branch is an external mutable pointer (`branches.yaml`) and a snapshot is merely a historical fact. One snapshot can belong to the history paths of several branches at once, and deleting a branch does not affect the snapshot itself.

The append-only design makes incremental backup and auditing easy.

#### 3.3.1 Deriving the event type

LFV does not store an explicit "event type" field in a Snapshot; it **derives** it from the `(parent, path, object)` triple and its difference from the parent snapshot:

| parent path | parent object | current path | current object | Event type | Rendering in `log` |
| ---- | ---- | ---- | ---- | ---- | ---- |
| — (first) | — | P | O | **add** (first snap) | `A  P` |
| P0 | O0 | P0 | O1 | **modify** | `M  P0` |
| P0 | O0 | P1 | O0 | **rename** | `R  P0 -> P1` |
| P0 | O0 | P1 | O1 | **rename + modify** | `R+M  P0 -> P1` |
| P0 | O0 | P0 or P1 | **null** | **delete** | `D  P0` (rendered `D  P1` when P1 ≠ P0, i.e. the last known on-disk path) |
| P0 | **null** | P1 | O1 | **revive** | `+  P1` |

Combinations that cannot occur — rejected on the write side, reported as a data-integrity error on the read side:

- A first snapshot with `object = null`: when a file is deleted before it has any snapshot, `lfv snap` only cancels its tracking and produces no snapshot (spec §4.3).
- Parent object `null` and current object `null`: a repeated deletion. Once the deletion snapshot has been written, the status-table row is gone and snap will not process that file again.
- Parent and current path and object all identical: spec §4.3 refuses an "unchanged" snap, and there is no `--allow-empty`.

A `revive` row can only be produced by the seam of `lfv relink` (§4.7); the `lfv revive` command is implemented as a rewind (§4.10) and appends no snapshot. If the O1 at a relink seam has appeared on the dst branch outside a contiguous run, it is likewise rejected as a loopback (§4.13).

> The `track` action only registers a `file-id` and the tracking flag; it **produces no Snapshot**. The first entry of the Snapshot chain is produced by the user explicitly running `lfv snap`, and its event type is derived as `add` (`parent=null`, `object≠null`). `track` and the first `snap` are two independent steps, and the same holds for auto-track.

> The benefit of not storing the event type explicitly: every Snapshot is a complete picture of "what the file was like at this moment", and adding a new event type requires no new field and no migration of old records. The drawback: the event type must be derived on the read side, so the rendering logic needs unit tests covering every combination.

> **On "why a Snapshot id is a ULID rather than a content hash"**:
> git uses a commit's content hash as its id, which is inherently tamper-evident but makes ids unreadable and unsortable by time. LFV chooses ULID as the id because:
> 1. In the single-file scenario, users frequently need to pick a snapshot out of `log` output by eye in chronological order; a ULID is far more readable than a bare hash;
> 2. Tamper resistance is delegated to a separate `digest` field, which is semantically clear and lets the `verify` command check it specifically;
> 3. With the id decoupled from the digest, adding or removing metadata fields later will not make the id drift.

#### 3.3.2 The canonical digest form

This is a cross-version compatibility promise and is independent of the whitespace and field order of the line on disk:

1. Take every field except `digest` and assemble a JSON object in the declaration order of the struct in §2.3: `id, parent, path, object, size, created_at, author, message`. For a tree snapshot, `path` and `size` participate as `null`, so both planes share one canonical form.
2. Compact serialization: no whitespace; strings escaped by `serde_json`'s default rules (only `"`, `\`, and control characters are escaped, non-ASCII stays as raw UTF-8); `None` → `null`; integers in decimal.
3. `digest = "blake3:" + hex(blake3(bytes))`.

`verify` recomputes it from the parsed struct and compares it with the `digest` in the record.

### 3.4 Tree Snapshot format

**Tree Snapshot** (`.lfv/trees/snapshots.log`, one JSON object per line):

```json
{
  "id": "snap:01HABC...",
  "parent": "snap:01HABZ...",
  "path": null,
  "object": "blake3:7fa3bc9d...",
  "created_at": "2026-05-17T10:00:00Z",
  "author": "goosy",
  "message": "chapter 3 complete",
  "digest": "blake3:fedcba98..."
}
```

Differences from a File Snapshot:
- The `path` field is empty (a tree needs no path to locate).
- `object` may not be null (a tree snapshot always points at an object — when "the whole working directory is empty" the manifest is zero bytes, its blake3 hash is computed as usual, and that is a valid tree-id).
- There is no `size` field (it participates as `null` in the canonical form).

### 3.5 Tree Object format

A **Tree Object** (stored in `objects/`, content-addressed) uses a YAML block sequence format, one line per entry:

```yaml
- "docs/ch1.md": blake3:abc123...
- "docs/ch2.md": blake3:def456...
- "docs/ch3.md": blake3:ghi789...
```

#### 3.5.1 Format specification

- **Structure**: a YAML block sequence, each line a single-key mapping.
- **Key** (path): always double-quoted. Paths contain only the Unix separator `/`; per the spec §4.2 path rules a trackable path contains no `"`, `\`, or control characters, so no escaping whatsoever is needed inside the key.
- **Value** (hash): the `blake3:` prefix plus lowercase hexadecimal, unquoted (it contains no YAML special character).
- **Ordering**: entries are in strictly ascending lexicographic path order (UTF-8 byte order), and duplicate paths are not allowed.
- **Empty manifest**: when there is no valid tracked file in the working directory, the Tree Object's content is the empty byte string (zero bytes); its blake3 hash is computed as usual and is a valid tree-id.
- **Nothing extraneous**: no comments, no blank lines, no BOM, and exactly one newline (`\n`) at the end of the file; an empty manifest is zero bytes with no newline at all.

#### 3.5.2 Canonicalization and tree-id

blake3 digests the byte string above directly, yielding that Tree Object's identity (the tree-id). Because the format is fully determined (quoting, ordering, newlines), identical working-directory content necessarily produces a byte-identical serialization and therefore the same tree-id — no extra canonicalization step is needed, since the storage format is the canonical format.

Two working-directory states with identical content — reached whenever and however — produce the same Tree Object hash, and `objects/` holds only one entry.

#### 3.5.3 Storage and compatibility

A Tree Object is usually very small (one line per tracked file), below the `min_bytes` compression floor, and is stored as `.raw`.

The format is a valid YAML subset, so external programs can read it with a standard YAML parser without knowing any LFV-internal convention. LFV itself can also extract entries line by line with the single regex `/^- "(.+)": (\S+)$/` rather than depending on a full parser.

**Comparing two Tree Snapshots**: compare their Tree Object manifests entry by entry. Same `path` and same `object` hash — unchanged; same `path` but different `object` hash — modified; present in only one of the manifests — added or deleted.

### 3.6 Mutable index and the `index.db` schema

The mutable index lives in `index.db` and records the working-area state of each file in the current working directory plus branch/tag caches. It is a rebuildable cache of current state and is not part of the immutable history objects. Storage objects and the Snapshot chain cannot be tampered with; together with the yaml / HEAD files under `.lfv` they form the repository's source of truth, and `index.db` can be rebuilt from that source of truth whenever it goes wrong.

`index.db` holds only **rebuildable** cached data; every source of truth lives in the filesystem (the Snapshot chain, the yaml files). The tables:

| Table | Description | Rebuildable |
| ---- | ---- | ---- |
| `tracked` | Tracked files that currently own a path (including `D` rows that have disappeared from disk and are waiting for `lfv snap` to record the deletion). Primary key file-id; caches the HEAD snapshot's path/object/size and the on-disk mtime/size/hash from the last scan, to short-circuit incremental scans. | ✓ |
| `untracked` | The dynamic untracked list in `config.yaml` ∩ LFV-visible ∩ paths that exist on disk; keeps the known former file-id for `lfv track` to reuse. | ✓ |
| `scan_meta` | Scan metadata: `last_completed_at` and the mtimes of `.lfvignore` and `config.yaml`, for incremental-scan invalidation. | ✓ |
| `branches` | Branch pointer cache per file. Source of truth: `.lfv/files/<ULID>/branches.yaml`. | ✓ |
| `tags` | Tag cache per file. Source of truth: `.lfv/files/<ULID>/tags.yaml` and `.lfv/trees/tags.yaml`. | ✓ |
| `snap_locator` | A bare `snap:<ULID>` → the file-id it belongs to (NULL for the tree plane). Source of truth: the `snapshots.log` files. | ✓ |
| `tree_file_refs` | Reverse-reference cache on the tree dimension. Source of truth: `.lfv/trees/snapshots.log` + `objects/`, rebuilt by `rebuild-index` and written by `lfv snap --tree`. Used to attach tree association information when rendering `lfv log <FS-ish>`, and by `lfv rewind <TS-ish>` to map manifest entries to file-ids. | ✓ |

> [!note] Notes on tracked / untracked
> - `tracked` **normally registers only files that are LFV-visible and currently exist in the working tree** (paths where `stat` succeeds and that do not match `.lfvignore`), with `present = 1`;
> - **Exception**: an entry that is tracked, has disappeared from disk, and is waiting for `lfv snap` to record the current filesystem state is kept with `present = 0` and shown as `D` in `status` output (§4.5);
> - an `untracked` row corresponds to a path in the dynamic untracked list of `config.yaml` that exists on disk; once it disappears from disk the row is deleted (the `config.yaml` list may stay);
> - the primary key is the file-id rather than the path: a rename pending write (`R`) has "current path ≠ HEAD path", and both that and `D` rows need to be located by file-id. The partial unique index `tracked_path` guarantees one row per path for files that exist on disk.
> - for a file that is tracked and still on disk, `path → file_id` lets the CLI resolve paths; once it has disappeared from disk it can still be located by `file-id`.

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

**Writing and rebuilding `tree_file_refs`**: when `lfv snap --tree` creates a Tree Snapshot it expands the Tree Object manifest and inserts one row per file (at that moment each entry's file-id comes straight from its `tracked` row); `rebuild-index` rebuilds it using the attribution rule in §3.10. It is used to attach tree association information when rendering `lfv log <FS-ish>`:

```
snap:01HXYZ  M  docs/note.md   "add chapter 2"
             └─ tree: "chapter 3 complete" [tree:v1.0]
snap:01HWWW  M  docs/note.md   "fix typo"
             └─ tree: "routine archive 2026-05-30"
```

### 3.7 Non-rebuildable state files

The following files record **user intent** or the state of an operation in progress, cannot be derived mechanically from the Snapshot chain, and are not overwritten by `rebuild-index`:

**`.lfv/files/<ULID>/HEAD`**

One per tracked file, containing the name of the current branch (plain text, one line):

```
main
```

LFV does not support a detached HEAD — `rewind` always creates a new branch to preserve the old HEAD, so HEAD always points at a named branch and is never a bare snap\_id.

**`.lfv/files/<ULID>/meta.yaml`**

```yaml
created_at: 2026-05-17T09:21:33Z
initial_path: "docs/note.md"      # path at track time; the only record of it before the first snapshot
retired: ~                        # ~ until `lfv relink <this> --onto <onto>`, then a nested block:
                                   #   at: 2026-06-01T08:00:00Z
                                   #   onto: file:01HA7BCD...
```

`initial_path` lets a file with no snapshot yet return to the `tracked` table after `rebuild-index` instead of being assigned a new file-id; `retired` keeps a retired file-id from competing for the path with the `onto` side during a rebuild.

**`.lfv/files/<ULID>/REPLAY.yaml`**

The in-progress state of a merge / rebase (structure in §4.14), existing only between `--continue` / `--abort`. While this file exists, other commands that would modify that file's history refuse to run, **including starting a new `lfv merge`/`lfv rebase`/`lfv merge --pick` on the same file-id**: a file-id can have at most one merge/rebase/pick in progress at a time, and a new one cannot start until the current one is finished with `--continue` or given up with `--abort`.

**`.lfv/trees/HEAD`**

The current head of the tree layer, containing `snap:<ULID>` or nothing (when the repository has no Tree Snapshot yet):

```
snap:01HABC...
```

**`.lfv/lock`**

A process-level advisory lock (`std::fs::File::try_lock`, Rust ≥ 1.89). It is acquired by `Repository::open`, and failing to acquire it is a fatal error — two `lfv` processes operating on the same repository at once is not supported.

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

`lfv config <key> [value]` reads and writes with dot-separated keys (`user.name`, `rename.autodetect`).

### 3.8 Branch and tag files (source of truth)

The yaml files below are the source of truth for branches and tags; the `branches` and `tags` tables in `index.db` are their rebuildable cache.

**`.lfv/files/<ULID>/branches.yaml`**

key = branch name, value = the `snap:<ULID>` at that branch's current HEAD:

```yaml
main: snap:01HXYZ...
rewind/7RQ2M9KA/1: snap:01HABC...
```

- Written when a branch is first created; the corresponding value is updated after `lfv snap` produces a new snapshot on the current branch.
- `lfv branch-delete` removes the corresponding key; the snapshots themselves are unaffected (append-only).

**`.lfv/files/<ULID>/tags.yaml`**

key = tag name, value = `snap:<ULID>`. Once created, a tag cannot be repointed; after deletion the same name can be created again:

```yaml
v1.0: snap:01HXYZ...
stable: snap:01HWWW...
```

**`.lfv/trees/tags.yaml`**

The tag table of the tree layer: key = `"tree:<name>"`, value = `snap:<ULID>`. Tree tags are globally unique and likewise cannot be repointed, but can be recreated after deletion:

```yaml
tree:v1.0: snap:01HABC...
tree:release: snap:01HZZZ...
```

User-input tag names may not contain `:` (used for namespace isolation); the `tree:` prefix is added by LFV automatically. The `tree:detour/<anchor-short>/<n>` tags applied automatically by `rewind <TS-ish>` also live in this table.

All yaml / HEAD writes go through `util::atomic_write` (a tmp file in the same directory + rename; `rename` can overwrite on Windows).

### 3.9 Write order and crash consistency

The write order of one `snap` (`ops::snap_file`):

1. `ObjectStore::put_*` (idempotent, repeatable)
2. `SnapshotLog::append` (append + fsync)
3. atomic replacement of `branches.yaml` (which also writes the branch itself on the first snapshot)
4. the `index` transaction: `tracked`, `branches`, `snap_locator`

The consequence of a crash after each step: after 1 → one unreferenced object more (reclaimed by `gc`); after 2 → one dangling snapshot (reclaimed by `gc --purge`); after 3 → the index lags behind the source of truth, repaired by `lfv rebuild-index`. The `ops` layer guarantees that 3→4 happen inside one function and that writes to the source of truth always precede writes to the index — the index can only lag, never lead.

### 3.10 The `rebuild-index` algorithm

1. Rebuild the schema.
2. Walk `files/*/`: read meta, HEAD, branches, tags, and the log. Write `branches`, `tags`, `snap_locator`. A `retired` file-id gets no `tracked` row (but still gets `snap_locator`, since `lfv log file:<src>` must keep working). According to the HEAD snapshot of the current branch:
   - present with `object != null` → a `tracked` row: `path = head.path`, `present = stat succeeded`, `head_*` filled in, `disk_*` left empty (forcing the next scan to re-hash);
   - present with `object == null` → deleted, no row;
   - no snapshot → `path = meta.initial_path`, `present = stat`, `head_* = NULL`.
3. Tree plane: read `trees/snapshots.log` → `snap_locator(file_id = NULL)`; `trees/tags.yaml` → `tags`; rebuild `tree_file_refs` with the attribution rule below.
4. `untracked`: `config.untracked ∩ visible ∩ exists on disk`; `file_id` is the last non-retired file-id that held that path in history (may be empty).
5. Clear `scan_meta` → the next scan is a full one.

**The `tree_file_refs` attribution rule**: a Tree Object holds only `{path, hash}` and no file-id. For each manifest entry `(P, H)`, search all file-id logs for a snapshot with `path == P && object == H && created_at <= tree.created_at` and take the file-id of the one with the **latest created_at**. This rule gives the right answer both for "a new file at the same path after a deletion" and for relink (the copied chain has newer ULIDs and later timestamps); the only residual ambiguity is two file-ids holding the same `(P, H)` at the very same moment, which normal operation does not produce.

### 3.11 Reachability and gc

- Roots: file plane = every branch head of every file-id (including retired ones) + every tag + the snapshots referenced by an existing `REPLAY.yaml`; tree plane = `trees/HEAD` + every `tree:` tag.
- Reachable snapshots = the closure from the roots along `parent`. Reachable objects = the `object` of every reachable snapshot ∪ the entries in the manifests of reachable Tree Snapshots.
- `lfv gc`: deletes objects that **no snapshot (dangling ones included) references**; the logs are left alone.
- `lfv gc --purge`: first `SnapshotLog::prune` each log according to reachability: unreachable snapshot lines are dropped whole and the kept lines stay byte-for-byte unchanged — the only change to a log besides appending, which deletes but never modifies an existing record; then delete objects according to the new reference set. A crash between the two steps is safe (it only leaves extra objects behind).

## 4. Internal Mechanisms

### 4.1 Command execution skeleton

```
cli::<cmd>::run(args)
  → repo::Repository::open(cwd)          // find .lfv upward; take lock; load config
  → ops::scan::lazy(&repo, mode)?        // only for commands in §4.3.3
  → resolve::*(&repo, args)              // typed targets (§2.8)
  → ops::<flow>(&mut repo, targets, opts) -> Report
  → cli render(Report)
```

### 4.2 `.lfvignore` and `config.yaml`

Tracking everything by default would also pull temporary files, build artifacts, and the like into the scan; `.lfvignore` and `config.yaml` provide "exclusion lists" at different levels, with different responsibilities:

| | `.lfvignore` | `config.yaml` (dynamic untracked) |
| ---- | ---- | ---- |
| Nature | static, repository-level, **highest priority** | a dynamic policy the user maintains through `lfv untrack` / `lfv track` |
| Status table | **never enters** `tracked` / `untracked` | registered as `untracked` while it **exists** on disk; the status-table row is **deleted** once it disappears (the config list stays) |
| Working-tree scan | **invisible**: pruned and skipped, taking no part in OS create/delete detection | **visible**: takes part in §4.3; a status-table row is maintained only for paths that exist on disk |
| `lfv track <file>` | **errors** on a match; `.lfvignore` must be edited | removed from the untracked list and brought into tracking |
| `lfv untrack <file>` | **errors** on a match | written into config; registered as `untracked` if it exists on disk |
| `lfv status --include-untracked` | **never appears** | **the only source** (the `untracked` rows of the status table, all of which come from config) |

Additional conventions:

- the `.lfv/` directory itself is always implicitly treated as an `.lfvignore` rule and never enters the status table;
- a path matching `.lfvignore` is, as far as LFV is concerned, equivalent to **not existing**: it is not auto-tracked, not untracked, and not listed by `--include-untracked`;
- during `rebuild-index`, the `untracked` rows = the dynamic list in `config.yaml` ∩ LFV-visible paths ∩ paths that **exist on disk**;
- `.lfvignore` syntax is a subset of gitignore syntax, only one file at the repository root is recognized, and matching uses the gitignore matcher of the `ignore` crate; directory patterns take part in traversal pruning;
- the dynamic untracked list records paths; after an untracked file is renamed on disk the new path is a trackable candidate and will be auto-tracked.

#### 4.2.1 Working-area access layer (`scan::fs`)

Every operation that reads or writes the working area by `RepoPath` goes through this layer (`ops` has it resolve a `RepoPath` into an on-disk path before calling `object::restore_to` and the like).

- **Normal form**: a `RepoPath` is always NFC (`unicode-normalization` crate). The scan converts on-disk file names to NFC; paths, branch names and tag names typed on the CLI are converted to NFC while parsing.
- **Locating** (`RepoPath` → on-disk path): resolve one path component at a time, trying the NFC form, then the NFD form, and when neither exists, list the parent directory and compare entries by their NFC form. Results are cached for the current process only and never written into `.lfv`: `.lfv` is copied across devices, and a map of on-disk names means nothing on another machine.
- **Writing**: when an existing entry is located, write to that entry; otherwise create it under the NFC name (missing parent directories are also created under NFC names).
- **Normalization collisions**: two on-disk names in one directory that are identical in NFC → error (spec §4.2). The scan finds them while walking a directory; the entry-by-entry comparison during locating finds them too.
- **Case collisions**: when writing path P, if the same directory already holds an entry differing from P only in case and a `stat` of P hits exactly that entry (so the directory is case-insensitive), report an error suggesting that case sensitivity be enabled (spec §4.2). Two active paths that are equal after case folding, found during a scan, are checked the same way. Detection is per directory by probing rather than inferred from the platform, since Windows can enable case sensitivity per directory.
- **Path too long**: when the underlying I/O fails because a path is too long, turn it into an error that states the cause (spec §4.2).

### 4.3 Working-tree scan (lazy scan)

Several commands (`lfv status`, `lfv track`, `lfv snap`, and so on) trigger a working-tree scan before executing, comparing the disk with `index.db`, updating `tracked` / `untracked`, and driving the automatic actions of §4.4–§4.6. Implemented as `scan::Scanner::run(Mode::{Incremental, Full})`.

#### 4.3.1 Scan actions

A few concepts first:

- Dynamically untracked: a path is in the dynamic untracked list of `config.yaml`.
- Trackable candidate: a path is not in the dynamic untracked list of `config.yaml`.
- State update:
  - if LFV-visible and dynamically untracked: set to `untracked` status;
  - if LFV-visible and already having a tracked row: compare `disk_mtime_ns` / `disk_size` → unchanged means the existing `disk_hash` is kept, changed means the hash is recomputed → compare with `head_object` / `head_path` to obtain `modified` or `unmodified` (status flags in §2.7); the chain of checks does not have to run to completion before a result is known;
  - if LFV-visible, unregistered, and a trackable candidate: perform auto-track (§4.4);
  - if not LFV-visible: delete the corresponding record from the status table, leaving `config.yaml` and the historical Snapshots untouched, and produce no `D`.

| Level | Subject | Scan action |
| ---- | ---- | ---- |
| **A. Registered paths** | rows already in `tracked` / `untracked` | For each row, first decide from the `.lfvignore` mtime whether LFV visibility has to be re-evaluated; if invisible, delete the status-table record, and if visible, `stat` + state update. A tracked row whose `stat` fails goes to auto-delete (§4.5). Cost O(number of registered paths). |
| **B. Newly discovered paths** | LFV-visible paths seen during traversal that are not yet registered | A dynamically untracked file is set to `untracked`; a trackable candidate first goes through rename detection (§4.6), and if unpaired, auto-track (§4.4). |

All the scan actions above update the status table. In addition, `lfv track` and `lfv untrack` also update `config.yaml` along with the status table.

#### 4.3.2 Incremental scan and invalidation

`scan_meta` records the completion time of the last scan and the mtimes of the rule files. Scans are **incremental** by default, to avoid recursing the whole working tree on every command:

- **Invalidation** (if any of these holds, level B performs a controlled full traversal):
  - `scan_meta` is empty (the first scan, or after `rebuild-index`);
  - the mtime of `.lfvignore` is later than the value recorded in `scan_meta` (the traversal/pruning boundary changed);
  - the mtime of `config.yaml` is later than the recorded value (the `untracked` rows are then **reconciled** against the dynamic untracked list and LFV-visible paths: registered if present, deleted if not);
  - the user runs `lfv status --refresh` (or an equivalent forced refresh).
- **Otherwise (incremental)**:
  - for a **directory**: if its mtime ≤ `last_completed_at` and it is already registered in the scan → do not descend; this stacks with **directory pruning** by `.lfvignore` (a pruned directory is invisible to LFV);
  - for a **file**: if it is already in the status table and its mtime ≤ `last_completed_at` → skip the level-B "is this a new path" check (level A still runs `stat`).
- **At the end of a scan**: update `last_completed_at` (to the time the scan started) along with the mtimes of `.lfvignore` and `config.yaml`.

> [!note] Note
> An mtime-based incremental strategy may miss files when a copy did not preserve timestamps, or when the filesystem's precision is only whole seconds; `--refresh` is the fallback.

#### 4.3.3 When scans run

| Command / scenario | Scan |
| ---- | ---- |
| `lfv status` | incremental scan by default (§4.3.2) |
| `lfv status --refresh` | forced invalidation, then scan |
| `lfv track` (no argument), `lfv snap` (no argument) | scan first, then batch track / snap |
| `lfv snap <file>`, `lfv mv`, `lfv delete`, `lfv rewind`, `lfv switch`, `lfv snap --tree`, `lfv rewind <TS-ish>` | incremental scan (these commands need an accurate `modified` determination) |
| `lfv status --include-untracked` | does **not** change the scan; merely prints the `untracked` rows of the status table as well (spec §4.3.2) |
| `lfv log`, `lfv show`, `lfv diff <FO-ish> <FO-ish>`, `lfv list`, tag/branch queries | no scan |

### 4.4 Auto-track (new files)

When a scan finds a path in the working tree that is **LFV-visible** and not yet in the status table:

- if it matches `.lfvignore` → **ignore** it (not registered, §4.2);
- if it is in the dynamic untracked list of `config.yaml` → register it as `untracked` (§4.2);
- if the path violates the spec §4.2 path rules → do not register it; only issue a warning (the scan continues) with a hint to add it to `.lfvignore`;
- otherwise → treat it as an "OS create" and `track` it automatically: `tracked::TrackedStore::create(path)` allocates a `file-id`, creates the directory, and writes `meta.initial_path`, while a row with `status = modified` and no `head_*` is inserted into the `tracked` table.

**No Snapshot is appended immediately**; the first Snapshot is written by a later `lfv snap`.

When auto-track runs: `lfv status` (executed on the spot during the scan), `lfv track` (when tracking with no argument), and `lfv snap` (before a no-argument batch snapshot).

**Cancelling tracking**: `lfv untrack <file>` writes to the dynamic untracked list in `config.yaml`; when the path exists on disk, an `untracked` row is registered alongside it (keeping the file-id). For a file that already has history this guarantees no new Snapshot will be produced; untracking an auto-tracked file with no history only deletes the tracking cache, which likewise guarantees no Snapshot is appended.

While a path is in the config untracked list and exists on disk, a scan does not auto-track it; once it disappears from disk the status-table row is deleted, and when the file reappears it is registered as `untracked` again.

**A friendly hint for same-named files**: if a path once had a deleted `file-id` (i.e. the latest Snapshot for that path has `object = null`), then after auto-track has allocated a new `file-id` at the same path, `lfv status` appends a hint under that file's entry:

```
  A   file:01HA7EEE...   docs/old-note.md  [newly tracked]
                      note: this path previously existed as file:023BHCA1 (deleted)
                      to continue its history instead, run:
                        lfv relink file:01HA7EEE --onto file:023BHCA1
```

How the lookup works: `snap_locator` cannot be searched by path, so for a newly registered path LFV searches the file-id logs for "the last file-id that owned this path and whose HEAD object is null"; this happens only once, when level B discovers the new path. That result is **not cached**: the `tracked` table has no `prior_file_id` column, and every `lfv status` looks it up again. Files in this `A` state are few and the hint disappears after the first snap, so the cost of re-querying is negligible; a cache column would mean a schema change plus one more recomputation rule in `rebuild-index`, which is not worth it.

The hint appears only while auto-track has produced a new `file-id`; once that new `file-id` has its first Snapshot, it is an independent file and the hint disappears.

For an untracked file that has history, its latest Snapshot still points at some Object, so when the user manually runs `lfv track <file>` again, LFV first removes it from the dynamic untracked list in `config.yaml`, then brings it into tracking under the original `file-id` recorded in `untracked.file_id` (searching the logs when there is no record), and updates it to a tracked row in the status table as a cache. Note that this differs from the algorithm for `lfv track <file>` after a delete: the latter always produces a new `file-id` by default, unless the user explicitly runs `lfv revive`.

### 4.5 Auto-delete (OS-deleted files)

When a scan finds that the path of a **tracked file** has disappeared from the working tree with no evidence of a rename (i.e. automatic rename detection failed to pair it), this is treated as "the OS deleted the file". LFV does **not** append a Snapshot immediately; it sets the `tracked` row to `present = 0, status = modified`. Because the current path does not exist on the filesystem, `lfv status` shows it as `D`.

A subsequent `lfv snap` command lands that state update (§4.9).

The benefit of this design: before `lfv snap`, a `D` file is still in the tracking list, so the user still has the chance to reclassify it as a rename with `lfv mv file:<old> <new-path>` and avoid an accidental deletion.

### 4.6 rename / move (lfv mv)

**`lfv mv` performs path operations only**: `<src>` accepts a path or a file-id (a file-id is located by its current path, which suits files already gone from disk such as `D` rows); `<dst>` accepts a path only, not a file-id. It is for renaming/moving a file that still exists on disk (or that the OS has just moved). The main reason `<dst>` may not be a file-id is the mental confusion it would cause — the user might mistakenly believe the surviving file-id is the `<dst>` side. To splice history onto another file-id, use `lfv relink` (see §4.7).

- `lfv mv <src> <dst>` (`ops::mv`): migrates the tracked file corresponding to src to the dst path.
  - Before appending the snapshot, hash src's current content (which is dst's content after the move) and run the loopback check (§4.13): on a hit the whole command is refused, and at that point the file on disk has not been moved.
  - If the src path still exists in the working tree and dst does not, the CLI moves the file to dst first and then appends the snapshot (atomic semantics).
  - If the src path no longer exists (the OS or an editor moved it first) and the file is already at the dst path, `lfv mv` only "registers the snapshot" and does not touch the disk again.
  - The `object` of the new Snapshot is determined by the hash of dst's current content: identical to the parent snapshot renders as a pure `R`, different renders as `R+M`.
  - `message` defaults to `rename: <old-path> -> <new-path>`.
- **Automatic detection vs. manual registration** (`scan::rename_detect`):
  - Automatic detection applies only when the content hashes are **exactly identical** (`rename.autodetect`, on by default). A hash is computed for a newly discovered path only when a tracked row with `present = 0` exists; if `hash == head_object` and the pairing is **unique** (exactly one disappeared row and one new path share that hash) → that tracked row's `path` is changed to the new path and `present = 1`, with no new file-id created; `lfv status` shows it directly as an `R` row.
  - Zero-byte files take no part in detection; many-to-many cases (several disappeared rows and several new paths sharing a hash) are not paired and are listed separately as `D` and `A`, leaving the user to declare the move with `lfv mv`.
  - If the OS moved the file **and** its content changed as well, automatic detection fails — `lfv status` then shows both a `D` row (the old file-id vanished from its path) and an `A` row (a new file-id at the new path). After reviewing, the user declares the history continuation explicitly with `lfv relink <new-file> --onto <old-file>`.
  - Automatic detection can also be turned off (useful for bulk rename + edit scenarios, to avoid mispairing).
- History review: `lfv log <FS-ish>` renders a rename event in the form "`R` old-path -> new-path", alongside `A` (add) / `M` (modify) / `D` (delete) / `R+M` (rename+modify) (see §3.3.1).

### 4.7 History continuation (lfv relink)

`lfv relink <src-file-id> --onto <dst-file-id>` splices the history of src-file-id's **current branch** onto the end of dst-file-id's **current branch**, while the on-disk file corresponding to src-file-id becomes bound to dst-file-id from then on. **It operates only on the current branch of each of the two file-ids, touches no other branch, and performs no 4-way merge content integration.**

The command is designed for the scenario "a user mistake caused LFV to produce a new file-id" (typically: a rename plus a content change at the OS level made auto-track allocate a new file-id) and should not be used as a routine command. If src-file-id has no history yet, dst-file-id's history is left as it is and only the file-id binding is transferred.

- **src-file-id**: status `A`/`M`, a live file present on disk (the new file-id, still tracked)
- **dst-file-id**: status `D`, a file that has disappeared from disk (the old file-id, awaiting continuation), or a deleted file whose latest snapshot has `object = null`
- **The file-id that survives is the `--onto` side (dst-file-id)**, consistent with the direction of the preposition

Afterwards, src-file-id's current path and content are appended as the next Snapshot of dst-file-id's history; src-file-id's `tracked` row is re-hung onto dst-file-id (the `file_id` is swapped and `head_*` taken from dst) and produces no further snapshot. **src-file-id's `files/<src-file-id>/` directory and `snapshots.log` are preserved in full** (append-only, never deleted), and `retired: {at, onto: dst}` is written into its `meta.yaml`; src-file-id becomes a "retired" file-id, `lfv log src-file-id` still works, and `lfv list` does not list it by default.

**Case 1: src-file-id has no snapshot history**

src-file-id has only a tracking record and produced no Snapshot. Afterwards: the file at src-file-id's current path uses dst-file-id as its file-id. The event type is derived per §3.3.1 by comparing dst-file-id's last snapshot with the current path/object (usually `R+M` or `R`, or `+` when dst was deleted).

**Case 2: src-file-id already has snapshot history**

src-file-id has already produced several Snapshots (the chain being `snap_A1 -> snap_A2 -> ... -> snap_An`). Afterwards, the snapshot chain of src-file-id's current branch is copied and appended to the end of dst-file-id's current-branch history:

- each snapshot gets a fresh ULID (`snap_B1 ... snap_Bn`), and `digest` is recomputed over the new fields
- `snap_B1.parent` = dst-file-id's latest snapshot; `snap_Bx.parent` = `snap_B(x-1)`
- `snap_B1`'s event type is derived by comparing dst-file-id's last snapshot with `snap_B1`'s path/object; from `snap_B2` onward the derived results match those inside the original src-file-id and are unaffected
- the event type at the seam may show up as a jump across paths, which is the expected cost of the user actively declaring a history continuation
- before copying, each snapshot to be appended goes through the loopback check (§4.13): if any of them hits, the whole relink is refused and nothing is written
- likewise, the on-disk file corresponding to src-file-id is bound to dst-file-id from then on

### 4.8 lfv delete

`lfv delete <file>` (`ops::delete`): deletes the file if it exists on disk and sets the `tracked` row to `present = 0, status = modified`. On the next `lfv snap`, LFV consults the current filesystem existence of the target file: if the file does not exist, it appends an `object = null` Snapshot and deletes that `tracked` row.

From then on that file-id has no `tracked` row. Its last path is preserved in that Snapshot and still resolves to it (by falling back to the last non-retired file-id that held the path, see §2.8), but once a new file takes the path it resolves to the new file-id; `lfv list --deleted` can still display it by path.

The file's **history is preserved in full**: `lfv log` / `lfv show` / `lfv diff` still work. To "revive" it, use `lfv revive` (§4.10) or `lfv rewind <file> <snap>` directly; both automatically create a preserving branch (so the existing delete event is not destroyed).

### 4.9 `lfv snap`: internal execution flow

**Single file** (`ops::snap_file`):

1. Read the `tracked` row; `status = unmodified` → skip.
2. `present = 1`:
   - `object.put_path(path)` → `(hash, size)`;
   - `hash == head_object && path == head_path` → refuse (unchanged);
   - `head_snap` is `None` (`A`) → the first snapshot, `parent = null`;
   - otherwise `snapshot::loop_check` (§4.13) → refuse on a hit and print the rewind hint;
   - append `Snapshot { parent: head_snap, path, object: hash, size }`; advance the branch table; update the `tracked` row's `head_*` and `disk_*`, with `status = unmodified`.
3. `present = 0`:
   - `head_snap` is `None` (`A` and already gone) → produce no snapshot, delete the `tracked` row, and leave the `files/<id>/` directory as an empty shell (which gc can clean up);
   - otherwise append `Snapshot { parent: head_snap, path: the last known on-disk path, object: null, size: 0 }`; advance the branch table; delete the `tracked` row.
4. The write order follows §3.9.

**Batch** (`ops::snap_all`) = scan + call `snap_file` for every row with `status = modified`, one independent transaction per file, so a single failure rolls nothing else back, with a summary report at the end. `lfv snap -m <msg>` without `<file>` and an explicit `lfv snap --snap-all -m <msg>` are the same entry point: both apply one `<msg>` to every currently `modified` file; `--snap-all` merely makes the intent "batch, sharing one message" visible on the command line and is not a separate algorithm. `--snap-all` and `--tree` are mutually exclusive, and the CLI layer refuses to accept both (§4.11).

### 4.10 `lfv rewind <file>`, `lfv switch`, `lfv revive`

**`ops::rewind_file(file, target: SnapId, opts { ff_first })`** (spec §4.5):

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

Normal mode and loop mode share the step "preserve the old HEAD on a new branch" and differ only in the `kind` of that preserving branch (`rewind` / `detour`).

**`ops::switch(file, branch)`**: the file is `modified` → refuse; the target branch HEAD has `object != null` → `restore_to` the current path and refresh `tracked.head_*`; the HEAD has `object == null` → delete the file from the working tree and delete the `tracked` row; write the `HEAD` file.

**`ops::revive(file, target?)`**: `target` defaults to the last snapshot on the current branch with `object != null`; calls `rewind_file` with `kind = revive`; the content is written back to `target.path` (the last known path) and a `tracked` row is inserted (`present = 1, unmodified`). If that path is already occupied by another active file-id, the command refuses.

### 4.11 `lfv snap --tree`: internal execution flow

`--snap-all` and `--tree` are two mutually exclusive flags, and the CLI layer refuses both at once outright (spec §4.3.3); `--snap-all` goes through `ops::snap_all` (the batch snap of §4.9, only explicitly named and requiring every `modified` file to share one `-m` message) and never enters `ops::tree_snap`. Once `lfv snap --tree` (`ops::tree_snap`) has passed the precondition of spec §4.3.3 (namely: no tracked file is in `modified` state at the time of the call, usually because `lfv snap` or `lfv snap --snap-all` has been run first):

1. Read `head_path` / `head_object` from all `tracked` rows with `present = 1, status = unmodified` (without reading the per-file logs) and build the Tree Object manifest (sorted by path).
2. Compute the blake3 hash of the canonical serialization to obtain the tree-id; reuse the object if `objects/` already has it, otherwise write it.
3. Append a Tree Snapshot record to `.lfv/trees/snapshots.log`, with parent pointing at the current tree HEAD (null if that is empty).
4. Update `.lfv/trees/HEAD` to the id of the new Tree Snapshot.
5. If `--tag` was given, write the tag into `.lfv/trees/tags.yaml`.
6. Expand the manifest and insert one row per file into the `tree_file_refs` table of `index.db` (the file-ids coming from the `tracked` rows of step 1), and write `snap_locator`.

### 4.12 `lfv rewind <TS-ish>`: internal execution flow

Once `lfv rewind <TS-ish>` (`ops::tree_rewind`) has passed the pre-check of spec §4.5.1:

1. If the current tree HEAD has no tag and is not an ancestor of the target, write a `tree:detour/<anchor-short>/<n>` tag.
2. Update `.lfv/trees/HEAD` to the snap id of the target Tree Snapshot.
3. Read the target tree-snapshot's Tree Object to obtain the `{path, file-object-hash}` manifest, and map each entry to a file-id through `tree_file_refs(tree_snap_id = target)`.
4. Handle all files in the following three categories:

**Category A: files that are in the manifest and whose file-id is still active (has a `tracked` row)** — for each such file, with `target_hash` as the goal, perform an FF-priority rewind (`rewind_file(ff_first = true)`):
  - **FF path**: if some branch's HEAD object hash == `target_hash` → prefer the current branch, and otherwise the most recently created matching branch → `switch` directly, creating no new branch. Message: `[FF] docs/note.md → branch main`
  - **rewind path**: otherwise → perform a standard rewind, automatically creating a preserving branch. Message: `[rewind] docs/note.md → kept old HEAD on rewind/7RQ2M9KA/1`
  - After either path, if the file's current path ≠ the manifest path, move the on-disk file to the manifest path and update `tracked.path` (the next snap records an `R`).

**Category B: files that have a `tracked` row but are not in the manifest** (files added after the tree-snapshot and already snapshotted):
  → only delete the working-area file, leaving `config.yaml` alone. A later scan (§4.5) marks it as `D` automatically. Message: `[deleted] docs/new-chapter.md`

**Category C: entries that are in the manifest but whose file-id currently has no `tracked` row (untracked, deleted, or retired), or that `tree_file_refs` cannot map**:
  → only write the file's bytes back to the manifest path, modifying no status table and no config. A later scan takes over:
  - the original file-id was deleted: the same-named-file hint of §4.4 applies, and the user can splice the history with `lfv relink`.
  - the original file-id was untracked: it shows as `~` after the scan, and the user decides whether to `lfv track` it.
  Message: `[restored] docs/old-chapter.md`

5. Print a summary.

### 4.13 Loopback detection and handling

To guarantee that each file's history presents an **acyclic content-evolution DAG** on any single branch, LFV enforces the **single-branch object uniqueness** invariant. Detection is currently triggered by `lfv snap` and `lfv verify`; `lfv mv`, `lfv merge`, `lfv rebase`, and `lfv relink` obey it just as much when appending snapshots (for how they handle it see spec §4.7.4 and §4.7 of this document).

- **On any single branch, the same `file-object-hash` may form only one contiguous run of snapshots and may not reappear non-contiguously.** Contiguous snapshots sharing one object (a pure rename, a path changed back) collapse into a single node at the content layer; `null` takes no part in uniqueness but does break a run: `O1 → null → O1` is a non-contiguous reappearance of O1.
- **History integrity comes first**: intermediate history is never silently discarded or rewritten, and the user must run `rewind` explicitly to "skip" a duplicate stretch.
- **Branch cleanliness**: a loopback would pollute the current branch, so the intermediate history must be moved to a `detour/<anchor-short>/<n>` preserving branch, which the user can review and then decide whether to delete.
- **Consistent with the semantics of the `rewind` command**: `lfv rewind` exists precisely to "go back in time and create a branch automatically", and the uniqueness invariant is designed for exactly this scenario.

#### 4.13.1 Detection timing

**Triggered by `lfv snap`**: when running `lfv snap <file>` (or handling one file inside a batch `lfv snap`), after computing the new snapshot's `object` field and before writing to `snapshots.log`, `snapshot::loop_check(log, head, new_object)` is called: **walking up the parent chain of the current branch**, it first skips the contiguous prefix whose object equals the new snapshot's (the same run), and if it then meets an ancestor with the same object → it returns the first snapshot `ancestor_snap` of the run that ancestor belongs to.

- Returning `None` → append the new snapshot normally.
- Returning `Some(ancestor_snap)` → a **loopback event** is triggered: this `lfv snap` is refused and a hint is printed.

#### 4.13.2 Handling a loopback event

LFV does not rewrite history automatically; it prompts the user to complete the rewind with `lfv rewind`, preserving all intermediate history:

```
warning: content of docs/note.md matches ancestor snap:01HXYZ on branch main
         (object hash blake3:abc123...)
suggestion: run `lfv rewind docs/note.md snap:01HXYZ`
            this will create a new branch anchored before the duplicate,
            preserving all intermediate history as a detour branch.
```

After the user runs `lfv rewind docs/note.md snap:01HXYZ`, `rewind_file` recognizes loop mode (the working-area content == the target object, the target is a strict ancestor of HEAD, and the HEAD object ≠ the target object), and therefore:

1. Creates a branch `detour/<anchor-short>/<n>` pointing at the current HEAD (`anchor-short` is the last 8 characters of the old HEAD's ULID, with `n` incrementing on a name collision); the branch name `main` itself is untouched;
2. Creates a **new snapshot** on the `main` branch whose:
   - `parent` points at the **parent snapshot** of `ancestor_snap` (`ancestor_snap` is the first entry of the matching run, so its parent's object necessarily differs);
   - `object` equals the new file content (i.e. the very `object` that triggered the loopback);
   - `path`, `message`, and the rest are recorded as in a normal snapshot.
3. Points `main`'s HEAD at this new snapshot, with the working-area content unchanged (it is already the new content).

This way all the intermediate history on the original branch (every snapshot after `ancestor_snap` up to the current one) is preserved on the `detour/...` branch, while the `main` branch "skips" that stretch of history and continues directly from the earlier state, and **no non-contiguous duplicate object hash is produced on `main`**.

#### 4.13.3 Verification requirements for `lfv verify`

The `lfv verify` command must walk all branches of every file and check whether the same `object` appears non-contiguously on one branch. If it finds one, it reports a **data-integrity error** (not a warning), because normal operation should never produce it (a loopback event is already refused and the user steered towards `rewind`).

```bash
error: branch 'main' of file 'docs/note.md' contains duplicate object hash:
       snap:01HXYZ (object blake3:abc123...)
       snap:02HABC (object blake3:abc123...)
       This violates the single-branch object uniqueness invariant.
```

### 4.14 `merge::replay` (merge / rebase / --pick)

```rust
pub struct ReplayState {          // persisted as REPLAY.yaml while in progress (§3.7)
    kind: Merge | Rebase | Pick,
    file: FileId, branch: BranchName,
    original_head: SnapId, preserved_branch: Option<BranchName>,
    steps: Vec<Step { base: Option<ObjectHash>, theirs: ObjectHash, theirs_snap: SnapId, path: RepoPath }>,
    next: usize,
}
```

- Locating the co-referent ancestor (spec §3.3): build a `HashSet` of every ancestor object on the ours chain, then walk the theirs chain for the first hit; the object that hits is `base-object`, and the nearest holder on each side is `base-snap-ours` / `base-snap-theirs`.
- Each step: `merge::three_way(base, ours, theirs)` (`diffy::merge`, with the conflict-marker line prefixes substituted for the labels of spec §4.7.3); the content layer performs a three-way merge, and "4-way" refers to the two base snapshots being allowed to differ. The text/binary determination is shared with diff: no NUL byte and decodable as UTF-8.
- No conflict → `put_bytes` → `loop_check` → pause on a loopback (spec §4.7.4) → append → advance the branch → `next += 1`.
- Conflict → text: write conflict markers into the working area; binary: leave the working area alone and prompt for `--continue --ours|--theirs` → save `REPLAY.yaml` → exit with a non-zero code. `--continue`: read the working area (or the chosen side) as the new object and continue; `--abort`: point the branch back at `original_head`, `restore_to`, and delete `REPLAY.yaml` (the intermediate snapshots are left dangling for gc to reclaim; `preserved_branch` is kept).
- rebase = `rewind_file(base-snap-ours, kind = rebase)` first, then enqueue the theirs steps followed by ours' original steps; merge = enqueue the theirs steps only; `--pick` = enqueue a single step with `base = pick.parent.object` (empty content when the parent is null or the parent's object is null), skipping the co-referent-ancestor lookup entirely (§3.3's ancestor search is needed only by merge/rebase). Aside from how base is computed and the step count, all three share one replay: conflict markers, `REPLAY.yaml` persistence, loopback detection, and `--continue`/`--abort` are identical; a paused `--pick` is likewise continued or rolled back with `lfv merge --continue` / `lfv merge --abort` — `kind = Pick` is for status display only and introduces no new command.
- `--continue` / `--abort` without `<file>` (spec §4.7.3): scan `files/*/REPLAY.yaml`; exactly one → act on that file; several → error asking for `<file>`; none → error.

### 4.15 `lfv verify`

`ops::verify` runs the following in order and summarizes only after all of them have run (it does not stop at the first error):

1. Objects: `object.iter` + `open` to recompute hashes; both extensions present for one hash;
2. Snapshots: recompute the digest of every line (§3.3.2); `parent` exists in the same log;
3. The snapshots pointed at by the branch/tag tables exist; the `HEAD` branch exists in the branch table (or the log is empty);
4. Object uniqueness on each branch (§4.13.3);
5. Tree Object manifests are parseable and the objects their entries point at exist; the objects of Tree Snapshots exist;
6. Sampled consistency between `index.db` and the source of truth (the branch table, `snap_locator`); an inconsistency only prints a hint to run `lfv rebuild-index`.

### 4.16 `lfv import`: internal execution flow

`lfv import <archive>` (`ops::import`) and `lfv export` (`ops::export`, spec §4.8) are inverse operations, and neither touches the working tree or the status table:

1. Unpack/open the archive and locate its `<ULID>/` directory; if `files/<ULID>/` already exists in the current repository, error out and refuse (which should not happen normally, as ULIDs are globally unique), performing no partial import and no merge of two histories.
2. Parse the `snapshots.log` inside the archive, recomputing the `digest` (§3.3.2) of each line and comparing it with the record; a single mismatch rejects the whole import.
3. Walk every `object` hash the log references: the archive carries those File Objects with it (in the same structure as `.lfv/objects/`), and for each hash, skip it if the local `objects/` already has it (deduplicating by content, consistent with the write policy of §3.2), otherwise write it as-is (with no fresh compression decision, trusting the encoding in the archive; `verify` will check the hash afterwards).
4. Copy the archive's `meta.yaml`, `HEAD`, `branches.yaml`, `tags.yaml`, and `snapshots.log` wholesale into `.lfv/files/<ULID>/`.
5. Write `snap_locator(snap_id, file_id)` for every snapshot in that log, so `lfv log <snap-id>` works immediately; **do not** create a `tracked` row and do not modify `config.yaml` — that file-id is in a "has history, not activated" state, equivalent to a "retired" file-id in the relink of §4.7 but without `meta.retired` (it was not absorbed by some dst, it simply has not been materialized yet).
6. Print a summary: file-id, number of branches, number of snapshots, number of objects imported. Mention that `lfv revive <file-id>` can restore it to the working tree.

`lfv revive` applies equally to a file-id with no `tracked` row whose `HEAD` snapshot has `object != null` (the precondition of revive in §4.10 only requires "a non-null snapshot can be located", not that the file was previously tracked); if the restore path is already occupied by another active file-id, it is refused per §4.10, and the user can make room with `lfv mv` first or choose another path.

### 4.17 Flow → module mapping table

| Flow | Triggering command | Entry point | Main dependencies |
| --- | --- | --- | --- |
| Visibility rules §4.2 | every scan | `scan::visibility` | `repo` (`.lfvignore`, `config.untracked`), the `ignore` crate |
| Lazy scan §4.3 | the §4.3.3 table | `scan::Scanner::run` | `index`, `tracked`, `object` (hash) |
| auto-track §4.4 | scan | `scan::autotrack` | `tracked::TrackedStore::create`, `index` |
| auto-delete §4.5 | scan | `scan` level A | `index` |
| Rename detection §4.6 | scan | `scan::rename_detect` | `index`, `object` (hash) |
| mv §4.6 | mv | `ops::mv` | `resolve`, `object`, `snapshot`, `tracked`, `index` |
| relink §4.7 | relink | `ops::relink` | `snapshot` (chain copy), `tracked` (retired), `index` |
| delete §4.8 | delete | `ops::delete` | `index`, `util::fs` |
| snap §4.9 | snap | `ops::snap_file` / `ops::snap_all` | `object`, `snapshot::loop_check`, `tracked`, `index` |
| rewind / switch / revive §4.10 | rewind, switch, revive | `ops::rewind_file` / `ops::switch` / `ops::revive` | `snapshot`, `object::restore_to`, `tracked`, `index` |
| snap --tree §4.11 | snap --tree | `ops::tree_snap` | `object::TreeManifest`, `tree`, `index` |
| rewind <TS-ish> §4.12 | rewind tree: | `ops::tree_rewind` | `tree`, `index.tree_file_refs`, `ops::rewind_file` |
| Loopback §4.13 | snap / mv / verify / merge / rebase / relink | `snapshot::loop_check` | — |
| merge / rebase / --pick §4.14 | merge, rebase, --continue, --abort | `merge::replay` | `diffy`, `ops::rewind_file`, `snapshot`, `object` |
| verify §4.15 | verify | `ops::verify` | the whole storage layer |
| import §4.16 | import | `ops::import` | `snapshot` (digest check), `object` (deduplicated write), `tracked` (meta/branches/tags copy), `index.snap_locator` |
| gc §3.11 | gc | `ops::gc` | `snapshot::prune`, `object::remove` |
| rebuild-index §3.10 | rebuild-index | `ops::rebuild_index` | `index`, `tracked`, `tree` |
| log / show / diff / list / branches / tags | queries | `ops::query::*` | `resolve`, `snapshot`, `diff`, `index` |

## 5. Roadmap Implementation Order (cf. spec §7)

| Version | Commands | Required modules |
| --- | --- | --- |
| v0.1 | init / track / snap / log / status / show | `util repo object snapshot reftable tracked index scan resolve ops(snap, track, log, status, show)`; **the full `index` schema lands in v0.1 together with `rebuild-index`**, since otherwise the core promise of "rebuildable" cannot be verified later |
| v0.2 | diff / branches / rewind / switch | `diff`, `ops(rewind_file, switch)`; `mv` / `delete` / `revive` / `relink` are grouped into v0.2 as well (they depend only on v0.1 modules) |
| v0.3 | tag / export / import / gc / verify | `ops(gc, verify, export, import)` |
| v0.4 | merge / rebase / --pick | `merge` |
| v0.5 | tree plane | `tree`, `ops(tree_snap, tree_rewind)`, `tree_file_refs` |
