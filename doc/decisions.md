# LFV — Key Design Decisions

This document records **why** LFV is designed the way it is. Each entry explains the rationale behind a specific design choice, including accepted trade-offs and rejected alternatives.

## 1. Single-file scope

All history objects belong to a single file; so-called global snapshot operations are essentially just executing the single-file snapshot operation on multiple files one by one.

The entire purpose of LFV is to track files as independent units. Requiring global snapshots would introduce cross-file coupling again—which is exactly the difference between Git and LFV.

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

The LFV mental model is: "I have a directory, and every file in it has version history." A track-by-default policy fits this model. Requiring explicit opt-in for every file reverses the model and makes users responsible for remembering to run `lfv track` after every file creation. Forgetting means losing history.

Files are usually excluded through `.lfvignore` or `config.yaml`, but they serve different purposes:

- Paths matched by `.lfvignore` are effectively invisible to LFV. They do not enter the status table, are not scanned, and never appear in output.
- Untracked entries in `config.yaml` represent runtime management decisions and produce explicit feedback when tracking state changes.

Together, these two sources determine the tracking state reflected by the status table. The status table normally reflects files that currently exist on disk (except `D` entries), preventing large numbers of ghost rows and keeping `lfv status` aligned with the real working tree.

The distinction exists because ignored files and dynamically untracked files have different runtime and user-interface requirements.

## 5. Snapshot

A Snapshot is a version record, roughly corresponding to a Git commit.

Snapshot IDs use ULID for readability and chronological ordering. Tamper resistance is handled separately through the `digest` field and verified by `lfv verify`.

Content-addressed identifiers such as Git SHA hashes couple identity and integrity verification together, forcing a trade-off between opaque hashes and user-friendly identifiers. LFV separates these concerns: ULIDs are human-friendly, time-ordered, and content-independent, while `digest` provides integrity verification.

Within snapshots, ULIDs are fully decoupled from paths. Rename and move operations are recorded as immutable events, making the Snapshot chain the complete source of truth for both file location and file content.

Without a stable ULID-based identity, renames either break historical continuity or require a mutable alias table, causing the source of truth to become fragmented.

## 6. Deletion is just a special Snapshot value; history is never lost

`lfv delete` removes the file from disk and marks the file as modified in the status table. During `lfv snap`, if the file no longer exists, LFV appends a Snapshot with `object = null` and removes the file record from the status table. All historical Snapshots remain intact and can be restored through `lfv revive`.

Deletion in LFV is a state transition, not erasure. Erasing history would violate the append-only guarantee and make recovery impossible. Treating deletion as a Snapshot whose object is null keeps the model structurally consistent and requires no special storage-layer behavior.

The two-step process provides a correction window. While the file remains in `D` state, `lfv mv` can still reclassify the event as a rename instead of a deletion.

## 7. Snapshot topology is a directed tree, not a DAG

Each Snapshot has exactly one parent pointer, forming a directed forest. Branches diverge and evolve independently. There is no physical merge node with multiple parents.

The topology is intentionally decoupled from content merging. Rejecting a Git-style DAG does not prevent content-level merging or alignment.

### Why a directed tree (single parent)?

LFV rejects multi-parent DAG topology primarily to preserve simplicity and deterministic historical tracing.

- **Minimal storage and indexing**: `snapshots.log` only needs an optional `parent` field.
- **Minimal algorithms**: Topological traversal becomes simple O(N) linear backtracking.
- **History purity and readability**: Every snapshot has a unique ancestor path.
- **No dual-line ambiguity**: Git merges create multiple valid historical paths. LFV's single-parent model eliminates that ambiguity entirely.

### Directed Tree vs. DAG

| Dimension | Git DAG | LFV Directed Tree |
| --- | --- | --- |
| Topology | Commits may have multiple parents. | Snapshots have at most one parent. |
| Complexity | High. Requires graph traversal and merge-related logic. | Low. A branched forest of linked histories. |
| Integration | Merge commits create graph convergence points. | Rebase/content-level alignment without multi-parent nodes. |
| Readability | History can be polluted by unrelated merges. | History remains clean and uniquely traceable. |
| Target use case | Large collaborative software projects. | Single-user, single-file, local history tracking. |

## 8. `lfv mv` and `lfv relink` are separate commands

`lfv mv <src> <dst>` performs path operations only. `<dst>` cannot be a `file-id`. `lfv relink <f_src> --onto <f_dst>` is dedicated to history continuation.

Allowing `lfv mv <src> <f_dst-id>` would mix path manipulation with history continuation and create misleading expectations about which `file-id` survives the operation.

`lfv relink` also solves a second problem that `lfv mv` cannot: merging another file's history. A common case is when `<f_src>` contains a superset of `<f_dst>` and the user wants to preserve only one identity while splicing the full history chain together. This is a history-consolidation operation, fundamentally different from a rename.

## 9. Only two storage object types

The only immutable storage objects in the repository are Object and Snapshot. All other concepts are mutable indexes.

## 10. Dual compression thresholds

`min_bytes` (default 4 KiB) and `max_bytes` (default 16 MiB) handle files that are too small or too large for compression. `blake3` is always computed from raw bytes. Uncompressed objects are stored as `.raw`; compressed objects are stored as `.zstd`.

A single threshold cannot handle both edge cases. Very small files may grow after compression due to framing overhead. Very large files can create memory spikes during compression. Dual thresholds explicitly address both problems.

## 11. SQLite as the index

Metadata requiring random updates—status tables, branches, tags, and global file indexes—lives in `index.db`.

The database is only a cache. The source of truth remains file-based.

Status tables and branch pointers are mutable and require efficient indexed lookups and atomic updates. Plain-file approaches degrade significantly at scale. SQLite provides ACID transactions, efficient indexing, and single-file deployment without requiring a daemon or network service.

## 12. Small-tool philosophy

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
