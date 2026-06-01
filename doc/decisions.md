# LFV — Key Design Decisions

This document records **why** LFV is designed the way it is. Each entry explains the rationale behind a specific design choice, including accepted trade-offs and rejected alternatives.

## 1. Single-file scope

All history objects belong to a single file; so-called global snapshot operations are essentially just executing the single-file snapshot operation on multiple files one by one.

The entire purpose of LFV is to track files as independent units. Only allowing global snapshots while disallowing per-file snapshots would introduce cross-file coupling again—which is exactly the difference between Git and LFV.

## 2. Rewind never destroys history

Snapshots are append-only and always reachable. Any "go back in time" operation is implemented by **creating a new branch**, never making snapshots before HEAD unreachable.

Destructive undo is a constant source of user error and data loss. Turning every rewind into branch creation preserves history by construction. The cost (slightly more complex branch management) is negligible compared with the safety guarantee.

`snapshots.log` is never rewritten. This naturally supports backup (incremental copying always works), auditing (a verifier can replay from the beginning), and crash recovery (partial writes at the tail can be detected and trimmed without corrupting earlier entries).

In-place mutation provides no meaningful benefit while sacrificing all three properties.

## 3. Content addressing + deduplication

LFV follows the same storage principle as Git. The content-addressed key is the `file-id`, which serves as the persistent identity of a file and the only stable reference.

A `file-id` is assigned when a file is tracked and persists throughout the file's entire lifecycle, including after deletion. It remains the only valid reference after a file disappears or its path changes. Users and scripts need a stable handle for querying history (`lfv log`), restoring content (`lfv revive`), or continuing history (`lfv relink`).

Paths are convenience aliases for common usage. Any command that accepts `<file>` also accepts either a path or an `f_*` identifier. Both forms are supported everywhere so users are never forced to look up a `file-id` during routine work.

## 4. Tracking policy

Files inside the working directory should normally always be tracked. LFV performs incremental lazy scans before relevant commands: newly created files are automatically tracked; tracked files deleted by the OS are marked as modified and shown as `D`; the next `lfv snap` appends a Snapshot according to the current filesystem state.

The LFV mental model is: "I have a directory, and every file in it has version history." A track-by-default policy fits this model. Requiring explicit opt-in for every file reverses the model and makes users responsible for remembering to run `lfv track` after every file creation. Forgetting means losing history. Track-by-default aligns with expected behavior and is consistent with how backup tools work. Files that should be excluded are handled through `.lfvignore` (static exclusion list) or `lfv untrack` (dynamic per-file opt-out).

Files are usually excluded through `.lfvignore` or `config.yaml`, but they serve different purposes:

- Paths matched by `.lfvignore` are effectively invisible to LFV. They do not enter the status table, are not scanned, and never appear in output.
- Untracked entries in `config.yaml` represent runtime management decisions and produce explicit feedback when tracking state changes.

Together, these two sources determine the tracking state reflected by the status table. The status table normally reflects files that currently exist on disk (except `D` entries), preventing large numbers of ghost rows and keeping `lfv status` aligned with the real working tree.

The distinction exists because ignored files and dynamically untracked files have different runtime and user-interface requirements.

## 5. Snapshot

A Snapshot is a version record, roughly corresponding to a Git commit.

Snapshot IDs use ULID for readability and chronological ordering. Tamper resistance is handled separately through the `digest` field and verified by `lfv verify` (see `design.md §5.2`). `lfv verify` can validate each Snapshot without touching the ID.

Content-addressed identifiers such as Git SHA hashes couple identity and integrity verification together, forcing a trade-off between opaque hashes and user-friendly identifiers. LFV separates these concerns: ULIDs are human-friendly, time-ordered, and content-independent, while `digest` provides integrity verification.

Within snapshots, ULIDs are fully decoupled from paths. Rename and move operations are recorded as immutable events, making the Snapshot chain the complete source of truth for both file location and file content.

Without a stable ULID-based identity, renames either break historical continuity or require a mutable alias table, causing the source of truth to become fragmented.

## 6. Deletion is just a special Snapshot value; history is never lost

`lfv delete` removes the file from disk and marks the file as modified in the status table. During `lfv snap`, if the file no longer exists, LFV appends a Snapshot with `object = null` and removes the file record from the status table. All historical Snapshots remain intact and can be restored through `lfv revive`.

Deletion in LFV is a state transition, not erasure. Erasing history would violate the append-only guarantee and make recovery impossible. Treating deletion as a Snapshot whose object is null keeps the model structurally consistent and requires no special storage-layer behavior.

The two-step process provides a correction window. While the file remains in `D` state, `lfv mv` can still reclassify the event as a rename instead of a deletion.

## 7. Symmetry Between Tree Snapshots and File Snapshots

`lfv snap --tree` creates a **Tree Snapshot** in exactly the same way that `lfv snap` creates a File Snapshot: by appending an immutable event record to an append-only log. It is addressed by a ULID, has a single parent pointer, contains a `digest` field for tamper detection, and points to a Tree Object.

A Tree Object is a content-addressed JSON manifest — a path-sorted list of `{path, file-object-hash}` entries representing all tracked files whose current HEAD resolves to a valid (non-null) object. Tree Objects and File Objects are stored in the same `objects/` directory and deduplicated by content. Two identical workspace states produce the same Tree Object hash and therefore share the same physical object entry.

Tree Snapshots are stored in `.lfv/trees/snapshots.log`. They maintain their own branch and tag sets and are managed through `--tree` variants of existing commands (`lfv log --tree`, `lfv branches --tree`, etc.). A repository may contain no Tree Snapshots at all; the tree layer is entirely optional.

`lfv snap --tree` requires that all modified tracked files have already been snapshotted (either by running `lfv snap` first or by using `--snap-all`). This guarantees that the Tree Object is built from the current HEAD of every file and prevents silently creating stale tree snapshots.

**Why not automatically snapshot files inside `lfv snap --tree`?** File snapshots and tree snapshots are intentionally separate so that user intent remains explicit. A Tree Snapshot is a deliberate milestone. Automatically creating file snapshots as a side effect would pollute individual file histories with unintended entries.

## 8. Single-branch object-hash uniqueness invariant and cycle prevention

**Invariant**: on any single branch of a file's history, the same file-object-hash must appear at most once.

Without this invariant, a branch could revisit a previous content state, creating a backward edge in the content-layer graph (a cycle). Cycles make content-layer rendering ambiguous: a node appears at two points in the timeline, and its annotation (message, timestamp) becomes unclear.

**Mechanism**: when `lfv snap` computes a new content hash and finds it equal to an ancestor's object hash on the current branch, it refuses to append the snapshot and instead outputs:

```
warning: content of docs/note.md matches ancestor snap_2 on branch B
suggestion: run `lfv rewind docs/note.md snap_2` to re-anchor
            this will preserve the intermediate history as branch detour_B
```

The user then runs `lfv rewind`, which:
1. Renames the current branch to `detour_<original-branch-name>` — preserving the intermediate history intact.
2. Creates a new snapshot on the original branch name whose parent points to the **parent of the matching ancestor** (not the ancestor itself), so the new snapshot and the ancestor are distinct nodes in the snapshot chain even though their object hashes are equal.

**Why point to the ancestor's parent, not the ancestor itself**: if the new snapshot's parent were the ancestor, both would share the same object hash and the same parent — they would be indistinguishable nodes in the snapshot chain, and the rendering could not show a meaningful edge between them.

**Why warn instead of auto-rewind**: the intermediate history (detour) may be intentional and worth reviewing before archiving. Surfacing the decision to the user is consistent with LFV's general philosophy of not silently restructuring history.

**`lfv verify` enforcement**: the single-branch uniqueness invariant is a checkable property. `lfv verify` traverses each branch and reports any violation as a data integrity error, not merely a warning. A second occurrence of the same object hash on a branch indicates the cycle-prevention flow was bypassed, which should not happen under normal operation.

## 9. Topology View Design

Every Snapshot has exactly one parent pointer, forming a directed forest. This applies equally to File Snapshot chains and Tree Snapshot chains. Once branches diverge, they evolve independently. There is no topological merge point — that is, no physical Merge Snapshot with multiple parents. This differs fundamentally from Git's DAG model.

The Snapshot topology and content merging are intentionally decoupled. Maintaining a directed-tree Snapshot topology does not imply rejecting content-level merging or alignment.

As a result, LFV exposes both Snapshots and Objects to the user. Objects serve as the visible nodes, while Snapshot links describe inheritance relationships between them. From the user's perspective, the resulting view forms a DAG.

- **Snapshot layer (storage layer):** Always a directed tree. Every Snapshot has exactly one parent pointer. This layer represents the physical storage structure and never changes.
- **Content layer (derived layer):** Nodes are object hashes. Edges are projected from Snapshot parent relationships onto object identities. Without additional constraints, cycles could appear when a branch revisits a previously seen object hash. Decision 8 eliminates such cycles through the single-branch object-hash uniqueness invariant, making the content layer a DAG.

LFV deliberately rejects Git-style multi-parent DAG topology in order to preserve maximum simplicity at the storage layer and absolute determinism in historical tracing:

- **Minimal storage and indexing**: `snapshots.log` requires only an optional `parent` field. Snapshot history naturally becomes a set of physical forks in singly linked chains.
- **Minimal algorithms**: Topological traversal degenerates into simple O(N) linear backtracking, avoiding Git's more complex topology-sorting and multi-path reachability analysis.
- **Pure and readable history**: A directed tree guarantees that the ancestor path of every Snapshot is uniquely determined.
- **No ambiguity of competing mainlines**: After a Git merge, multiple historical paths coexist in the topology and there is no inherent notion of which path is the "true" mainline. LFV's single-parent topology eliminates this ambiguity entirely and allows different branches to evolve at different granularities.

### Comparison: Directed Tree vs. DAG Topology

| Dimension | Git DAG Topology | LFV Directed-Tree Topology |
| :--- | :--- | :--- |
| **Topology Definition** | A Commit may have one or more parents; multiple parents are used to represent merges. | A Snapshot may have at most one parent; the parent pointer is unique. |
| **Underlying Complexity** | High. Requires handling complex cross-path relationships and multi-path reachability analysis. | Low. The structure is a branched forest of singly linked histories, making storage, traversal, and garbage collection lightweight. |
| **Integration Mechanism** | **Topology-level Merge (Merge Commit)**: convergence is represented directly in the graph through multiple parent pointers. | **Reconstruction / Content-level Merge (Rebase / Content Merge)**: no physical convergence node exists; alignment is achieved through parent redirection or content continuation under a single-parent Snapshot. |
| **History Readability** | Easily polluted. The history of a single file (`git log <file>`) can be obscured by unrelated commits and crossing merge lines. | Completely clean. The ancestor path is uniquely defined, preserving the file's actual evolution over time. |
| **Target Scenario** | Distributed software projects with multiple collaborators, multiple files, and frequent merges. | Single-user, single-file, offline, local scenarios such as notes, configuration files, and NAS backups. |

### Why a Composite View?

LFV treats content merging and historical-topology merging as two separate concerns.

When users merge files, what they actually care about is whether the content has been unified, not whether multiple historical branches converge into the same Snapshot node. In the LFV interface, a merge is considered complete once the resulting Objects are identical. There is no need to force convergence at the Snapshot level.

Git, by contrast, expresses convergence through Commit topology itself. LFV intentionally separates content structure from historical structure. This separation greatly simplifies merge algorithms and makes the workflow easier for users to understand.

## 10. Tree-plane design principles

### 10.1 tree-id uses content hash, not ULID

The tree-id (the identity of a Tree Object) is the content hash `blake3(canonical manifest bytes)`, not an independent ULID like a file-id.

A repository logically has only "one tree" — the set of all tracked files in the working directory, which changes as content changes. It does not need a stable identity of "which tree this is". Using the content hash as tree-id provides natural deduplication (two identical workspace states share the same Tree Object) and avoids the overhead of maintaining a separate identity (ULID + meta.yaml) for trees. This differs from file-id design — files need a stable identity across renames and moves, hence a ULID decoupled from paths; trees do not need identity across content changes, so a content hash suffices.

### 10.2 Tree has no branches, only tags and HEAD

Tree Snapshot chains have no branch set. They have only: tags (prefixed with `t:`, globally unique), and a single HEAD pointer (the current Tree Snapshot).

The core value of branches is to support "multiple independent evolution lines of the same file" — a typical need on the file plane. Trees are milestone-style global snapshots; their usage pattern is linear progression, and they neither need nor benefit from multiple parallel evolution lines. Introducing tree branches would only increase cognitive load without meaningful benefit.

**Tree HEAD is stored in a separate file `.lfv/trees/HEAD`**: tree HEAD is a current state that cannot be reconstructed from the Snapshot chain (it records "which node the user is currently on in tree history", not which Snapshot is the latest). Other state in `index.db` (branch pointers, file HEAD) can be rebuilt from Snapshot chains during `rebuild-index`; tree HEAD, once lost, cannot be rebuilt, so it must be persisted independently of the rebuildable `index.db` to avoid accidental overwrite during `rebuild-index`.

`lfv switch <branch>` only operates on file branches; there is no tree switch operation (because trees have no branches). Tree position changes are done only via `lfv rewind t:<tag|snap-id>`.

### 10.3 tree rewind does not directly operate on config, branches, or status table

When `lfv rewind t:<tag|snap-id>` executes, it does only two things: update `.lfv/trees/HEAD`, and perform byte-level operations (write or delete) on working directory files. It does not directly call `track`/`untrack`/`revive` commands, does not modify the dynamic untracked list in `config.yaml`, and does not modify any branch pointers or the status table.

`config.yaml`, branches, and the status table belong to the responsibility boundaries of `track`/`untrack`/`delete`/`revive` commands. If tree rewind crossed these boundaries, users would find it hard to predict which commands have side effects on config, breaking command responsibility clarity.

After byte operations, the existing lazy scanning mechanism (§6.8) will naturally detect changes and drive state updates on the next `lfv status` or `lfv snap`. "Newly created files" and "disappeared files" produced by tree rewind are identical in effect to files directly manipulated by the OS, so the user's mental model does not need special casing.

### 10.4 tree rewind uses Fast-Forward preference per file

`lfv rewind t:<tag|snap-id>` does not directly execute rewind (which would create a new branch) for each file that needs restoration. Instead, it first checks whether a Fast-Forward path exists:

- **FF path**: If the current HEAD of a branch has an object hash equal to the target hash, directly `switch` to that branch without creating a new branch. Prefer the current branch (no switching cost); if the current branch does not match, pick the most recently created matching branch.
- **rewind path**: Otherwise, perform a standard rewind, automatically creating a new branch (`rewind/<snap-short>/<n>`).

Tree rewind is a batch operation that may affect dozens of files. If every file unconditionally created a new branch, the resulting branch noise would greatly interfere with users' reading of each file's history. The FF path covers most tree rewind scenarios in practice (because tree snapshots are typically created immediately after each file's snap, at which point each file HEAD's object is exactly the object recorded in the tree object), making zero branch pollution the norm. Only when truly rewinding to a non-HEAD position does a new branch get created, consistent with Decision 2 (Rewind never destroys history).

This is isomorphic to Git's Fast-Forward merge: no extra node is produced when FF is possible; branching happens only when necessary.

### 10.5 Tree Object uses YAML block sequence, not JSON

The Tree Object manifest uses YAML block sequence format (`- "path": hash`), not a JSON array (`[{"path":...,"object":...}]`).

**Rejected alternative**: JSON array. Canonicalization of JSON requires additional conventions (no extra whitespace, fixed key order, no trailing commas). Implementations must use a controlled serializer rather than plain `to_string()`, separating canonicalization requirements from the format specification, which easily leads to inconsistent hashes due to implementation omissions.

**Reasons for choosing YAML block sequence**:

**Storage format = canonicalization format**: Each line `- "path": hash` is fully deterministic (quotes, sorting, single newline). `blake3` digests the byte string directly, with no extra canonicalization step. The canonicalization constraint is embodied in the write rules, not an附加 serialization protocol.

**Safety of paths as keys**: In YAML, bare keys have special meanings for characters like `#`, `[`, `{`, `,`, `&`, `*`. The path character set conflicts with these. Double-quoted keys completely eliminate all special-character issues — including paths with spaces (very common on Windows/NAS scenarios). Paths are always stored with Unix separator `/`, not `\`. The only character that needs escaping inside the key is `"`, which almost never appears in real paths, so the overhead of quotes is negligible.

**External compatibility**: This format is a valid YAML subset; external programs can read it with a standard YAML parser. LFV internally can also parse line by line with a simple regex `/^- "(.+)": (\S+)$/`, without relying on a full parser. JSON also has external compatibility, but YAML is slightly better in character saving and parsing simplicity.

**Character efficiency**: Each entry saves about 20% of characters (~35 characters per YAML line vs. ~45 per JSON object). Tree Objects are usually below the `min_bytes` compression floor, stored as `.raw`, so character efficiency directly translates to storage efficiency.

## 11. Two storage object categories, each with two subtypes

The truly immutable storage objects in the repository fall into two categories — **Object** and **Snapshot** — each with two subtypes. Everything else is mutable index (see `design.md §4.2`).

| Category | Subtype | Contents | Addressing |
| --- | --- | --- | --- |
| Object | **File Object** | Raw file bytes (compressed) | `blake3(raw bytes)` |
| Object | **Tree Object** | Sorted `{path, file-object-hash}` manifest (YAML) | `blake3(canonical manifest)` |
| Snapshot | **File Snapshot** | Event record for one file | `snap_<ULID>` |
| Snapshot | **Tree Snapshot** | Event record for a working-tree state | `snap_<ULID>` |

Both Object subtypes are stored in the same `objects/` bucket directory, content-addressed, fully immutable. Both Snapshot subtypes are append-only and follow the same JSON Lines format, but live in separate directories:
- File Snapshot: `.lfv/files/<file-id>/snapshots.log`
- Tree Snapshot: `.lfv/trees/snapshots.log`

**Why a Tree Object now exists**: LFV's use cases include whole-directory milestoning (e.g. "complete draft of this book"). A Tree Object makes this a first-class operation while preserving the core invariant — content identity is determined by hash, not by snapshot id. Two identical working-tree states produce the same Tree Object hash and share one entry in `objects/`, just as two files with identical content share one File Object.

**Why Tree Object stores file-object-hashes, not file-snapshot-ids**: The node identity in LFV's history graph is content (object hash), not the metadata record (snapshot id). A Tree Object that references file-object-hashes is content-addressed end-to-end; referencing snapshot ids would couple the tree's identity to incidental metadata (message, timestamp), breaking deduplication and the content-node semantics.

## 12. Dual compression thresholds

`min_bytes` (default 4 KiB) and `max_bytes` (default 16 MiB) handle files that are too small or too large for compression. `blake3` is always computed from raw bytes. Uncompressed objects are stored as `.raw`; compressed objects are stored as `.zstd` (see `design.md §5.1`).

A single threshold cannot handle both edge cases. Very small files may grow after compression due to framing overhead. Very large files can create memory spikes during compression. Dual thresholds explicitly address both problems. `reject_if_larger` handles the fallback for already-compressed binaries (e.g., images) that provide no compression benefit. `blake3` is always computed from raw bytes, ensuring the hash is independent of storage format.

## 13. SQLite as the index

Metadata requiring random updates—status tables, branches, tags, and global file indexes—lives in `index.db`.

The database is only a cache. The source of truth remains file-based.

Status tables and branch pointers are mutable and require efficient indexed lookups and atomic updates. Plain-file approaches degrade significantly at scale. SQLite provides ACID transactions, efficient indexing, and single-file deployment without requiring a daemon or network service.

## 14. `lfv mv` and `lfv relink` are separate commands

`lfv mv <src> <dst>` performs path operations only. `<dst>` cannot be a `file-id`. `lfv relink <f_src> --onto <f_dst>` is dedicated to history continuation.

Allowing `lfv mv <src> <f_dst-id>` would mix path manipulation with history continuation and create misleading expectations about which `file-id` survives the operation, while `f_src` is retired.

`lfv relink` also solves a second problem that `lfv mv` cannot: merging another file's history. A common case is when `<f_src>` contains a superset of `<f_dst>` and the user wants to preserve only one identity while splicing the full history chain together, then retiring `f_src`. This is a history-consolidation operation, fundamentally different from a rename.

## 15. Tree reverse references use `tree_file_refs` DB table

Reverse references for which Tree Snapshots contain each file are stored in the `tree_file_refs` table in `index.db`, not in a separate file under each file-id directory. Fields: `tree_snap_id` (`snap_<ULID>`), `file_id`, `file_object` (blake3 hash). The source of truth is `.lfv/trees/snapshots.log` + `.lfv/objects/`; `rebuild-index` reconstructs the table.

**Rejected alternative**: Maintain a YAML file under each file-id directory (`.lfv/trees/.yaml`) with keys = `snap_<ULID>` and values = file-object-hash. This approach produces many small file writes when many files are involved, and lacks transaction guarantees (a crash during writing can cause inconsistency).

**Reasons for using the DB table**: `index.db` provides ACID transactions, efficient single-table queries, and the `tree_file_refs` table is purely a cache (the source of truth is in `.lfv/trees/snapshots.log` + `objects/`). Putting it in the db keeps semantics consistent with other rebuildable indexes; `rebuild-index` rebuilds it uniformly, without risk of missing reverse references for some file.

**Why snap_id as primary key (not tag name)**: Every Tree Snapshot has a message, and is meaningful even without a tag. Using snap_id as the primary key ensures that all Tree Snapshots have reverse references; when rendering, the tag is looked up dynamically from `.lfv/trees/snapshots.log`: if a tag exists, display the tag name; otherwise display the message. Both are meaningful. "Whether a tag is applied" only affects display, not the completeness of history.

## 16. Namespace isolation: user input must not contain `:`

Tree tags are prefixed with `t:` (internal management). Files have an implicit `f:` namespace (usually not displayed). **Users are not allowed to include `:` in branch names or tag names** when providing input. This achieves namespace isolation, preventing user input from colliding with internal prefixes.

**Why**: If users were allowed to enter `t:foo`, the CLI would not be able to distinguish whether the user intentionally references a tree tag or truly wants to create a file tag named `t:foo`. By forbidding `:` in user input, the ownership of namespaces is clear: prefixed references are always internally generated, unprefixed references are always user input.

## 17. Small-tool philosophy

CLI subcommands should remain clear, composable, and scriptable. LFV avoids premature abstraction toward graphical interfaces or services.

LFV is designed for automation, shell scripts, and integration with external tools. A GUI or daemon can always be built on top of a stable CLI; the reverse is much harder.

## Possible Future Extensions

- Simple HTML5 interface with visual branch-tree display.
- Hook system (e.g., automatic snapshots before save).
- Cross-repository object pool sharing.
- Bridges to Git LFS and NAS vendor APIs.

## Open Questions

1. **Rename auto-detection threshold** (`rename.autodetect`): exact hash match only, or similarity-based matching above a configurable threshold?
2. **Default restore point for `lfv revive`**: automatically restore the latest non-null object snapshot, or require explicit snapshot selection?
3. **History continuation for same-name new files**: should additional confirmation or `--force` be required when reviving history onto a newly created file with the same path?
