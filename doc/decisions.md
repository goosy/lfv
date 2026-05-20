# LFV — Key Design Decisions

> This document records **why** LFV is designed the way it is.

Each entry explains the rationale behind a specific design choice, including the trade-offs accepted and the alternatives rejected.

## 1. Single-file scope

All history objects belong to a single file; batch commands with no arguments simply execute the single-file operation on each file in sequence — there is no "global snapshot."

This follows directly from the project's founding motivation. The entire purpose of LFV is to track files as independent units. Allowing a global snapshot would reintroduce the cross-file coupling that `git` already handles — and that LFV deliberately avoids.

## 2. Rewind never destroys history

Any "go back in time" operation is implemented via **creating a new branch**, ensuring no snapshot before HEAD ever becomes unreachable.

Destructive undo is a constant source of user error and data loss. By making every rewind a branch creation, the invariant "snapshots are append-only and always reachable" is preserved unconditionally. The cost (slightly more complex branch management) is negligible compared to the safety guarantee.

## 3. Append-only history

`snapshots.log` is never rewritten.

Append-only files facilitate backup (incremental copy always works), auditing (a verifier can always replay from the beginning), and crash recovery (a partial write at the tail can be detected and trimmed without corrupting earlier entries). In-place mutation gains nothing and breaks all three properties.

## 4. Content addressing + deduplication

Even with massive duplicate content across thousands of independent files, the object store holds only one copy.

Per-file versioning without deduplication would produce catastrophic storage bloat (e.g. 1000 notes each versioned 100 times, even if most snapshots share content). Content-addressed storage makes deduplication zero-cost at the architecture level: two identical byte sequences simply resolve to the same object hash.

## 5. SQLite as the index

Metadata requiring random updates — status table, HEAD, branches, tags — lives in `index.db`; purely historical data lives in files.

The status table and branch pointers are **mutable** and need efficient random-access reads and atomic updates. A plain-file approach (e.g. one YAML per file) works at small scale but degrades badly with thousands of tracked files. SQLite provides ACID transactions, efficient indexed lookups, and a single-file deployment model — no daemon, no network. The historical Snapshot chain, by contrast, is append-only and sequential, so a flat JSON Lines file is simpler and sufficient.

## 6. Small-tool philosophy

CLI subcommands are clear, composable, and scriptable; no premature abstraction toward a GUI or service.

LFV is designed for automation, shell scripts, and integration with other tools (sync scripts, editors, CI). Prioritizing a clean CLI contract keeps the tool composable. A GUI or daemon layer can be built on top of a stable CLI; the reverse is painful. v1.0 deliberately excludes a GUI (see Non-Goals in `design.md §3`).

## 7. Only two storage object types

The only truly immutable storage objects in the repository are Object and Snapshot; there is no tree object. Everything else is mutable index (see `design.md §4.2`).

Git's tree object exists because a commit spans multiple files — a tree encodes "the directory listing of a commit." LFV's Snapshot corresponds to a single file and points directly to an Object, so an intermediate tree layer adds complexity with zero benefit. Keeping the object model minimal makes the storage layer, GC, and `verify` command simpler and easier to reason about.

## 8. Snapshot id separated from tamper-resistance

Snapshot ids use ULID for human readability and time ordering; tamper-resistance is handled by the dedicated `digest` field, verified by `lfv verify` (see `design.md §5.2`).

Content-addressed ids (like Git's SHA hashes) couple identity with tamper-resistance, which forces a trade-off: either ids are opaque hashes (bad UX) or they are human-readable strings that cannot serve as integrity proofs. LFV separates these concerns: ULID ids are paste-friendly, time-ordered, and content-independent; the `digest` field provides independent tamper detection. `lfv verify` can check every Snapshot without touching ids.

## 9. Path is a field on a Snapshot, not the file's identity

`file-id` is fully decoupled from path; rename/move is recorded as an **immutable event**, structurally identical to "content change." This makes the Snapshot chain the complete source of truth for both file location and content, eliminating the truth-source fragmentation that a mutable `aliases` list would cause.

If path were the file's identity, every rename would either break history continuity or require a mutable alias table. A mutable alias table splits the source of truth: history is in `snapshots.log`, but current identity is in the alias table — the two can drift, especially after crashes or partial syncs. By making path a field on each Snapshot and using a stable `file-id` as identity, the Snapshot chain becomes the sole source of truth for both file location and content. Rename, move, rename+modify, and delete are all just different field combinations on the same Snapshot structure.

## 10. Deletion = deletion marker; history is never lost

`lfv delete` removes the file from disk and marks its status-table entry as `modified`. `lfv snap` then checks the target file's FS state, and if the file does not exist, appends an `object=null` Snapshot and removes the file record from the status table. All historical snapshots are fully preserved and the file can be revived at any time via `lfv revive`.

Deletion in LFV is a **state transition**, not an erasure. Erasing history would violate the append-only guarantee (Decision 3) and make `lfv revive` impossible. Treating delete as a Snapshot with `object=null` is structurally consistent with the rest of the model — no special cases in the storage layer.

The two-step design (mark `modified` → confirm on `lfv snap`) gives the user a recovery window: while the file is still in `D` status, `lfv mv` can reclassify it as a rename instead of a deletion.

## 11. `file-id` is the only stable reference

All commands that accept `<file>` also accept a path or `f_*`. A `file-id` is assigned at track time and persists through the file's entire lifecycle — including after deletion — and is the only still-valid reference token after the file disappears or its path changes.

After a file is deleted or renamed, its path no longer resolves. Users and scripts need a stable handle to query history (`lfv log`), restore content (`lfv revive`), or relink history (`lfv relink`). Path is a convenience alias for the common case; `file-id` is the durable identity. Both are accepted everywhere so users are never forced to look up ids for routine operations.

## 12. Track-by-default policy

Files in the working directory are, under normal circumstances, always in a tracked state. LFV runs an incremental lazy scan before relevant commands: newly created files are auto-tracked; OS-deleted tracked files are flagged `D` and receive a deletion marker on the next `lfv snap`.

The mental model of LFV is "I have a directory; every file in it has version history." Requiring explicit opt-in for every file inverts this: the user must remember to run `lfv track` after every new file creation, and forgetting loses history. Track-by-default matches the expected behavior and is consistent with how backup tools work. Files that should be excluded are handled via `.lfvignore` (a static exclusion list) or `lfv untrack` (dynamic per-file opt-out).

## 13. Snapshot topology is a directed tree, not a DAG

Each Snapshot has exactly one parent pointer, forming a directed forest. Branches diverge and evolve independently; there is no topological merge point. To "realign" two branches, the user must explicitly run `lfv rebase` (not yet implemented) — branches never merge automatically.

Git's DAG topology exists to support merge commits, which record "two branches were integrated into one." LFV explicitly excludes automatic merging (see Non-Goals in `design.md §3`) because per-file history rarely needs it — and when it does, the user should decide the outcome explicitly. A directed tree is strictly simpler: the storage layer, index layer, and `log` rendering never need to handle multiple parent pointers. At the UI layer, file-hash (the object's blake3) serves as the measure of content identity, so `lfv log --graph` and `lfv branches` can show when two branches share the same content, while each Snapshot's identity (ULID) remains unique and independent. In LFV's single-user, single-file, local scenario this design incurs almost no cost.

## 14. Dual compression thresholds

`min_bytes` (floor, default 4 KiB) and `max_bytes` (ceiling, default 16 MiB) handle files that are too small or too large to compress; `blake3` is always computed over raw bytes; objects that skip compression or are rejected as ineffective are stored as `.raw`, compressed objects as `.zstd` (see `design.md §5.1`).

A single threshold cannot handle both boundary cases. Files below the floor may actually grow after compression due to frame overhead. Files above the ceiling require reading the entire file into memory, causing memory spikes. The dual thresholds address each case explicitly; `reject_if_larger` handles already-compressed binaries (e.g. images) where compression provides no benefit. Computing `blake3` over raw bytes ensures the hash is independent of the storage format.

## 15. `.lfvignore` is invisible; the status table normally reflects files that exist on disk

Dynamic untracked paths enter the table when they exist on disk and are removed when they disappear (the config entry may remain); `--include-untracked` lists only `untracked` rows in the status table. Tracked, missing, modified files are the exception and render as `D`. Scans are mtime-incremental by default with `.lfvignore` directory pruning (see `design.md §6.7, §6.8`).

Paths matching `.lfvignore` simply do not exist for LFV — they are not entered in the status table, not scanned, and not shown in any output. This matches `.gitignore` semantics and keeps the user mental model low-friction. The status table records only files that currently exist on disk (the `D` exception aside), avoiding a large number of "ghost rows" and ensuring `lfv status` output always reflects the real working tree state.

## 16. `lfv mv` and `lfv relink` are separate commands

`lfv mv <src> <dst>` handles path operations only; `<dst>` is not allowed to be a `file-id`. `lfv relink <f_src> --onto <f_dst>` handles history continuation exclusively.

If `lfv mv <src> <f_dst-id>` were allowed to mix path operations with history continuation, users would naturally assume that the surviving `file-id` after the operation is `<dst>`'s — but in a relink the surviving id is the `--onto` side (`f_dst`), while `f_src` is retired. This mental model mismatch almost inevitably causes misuse.

`lfv relink` also solves a second problem that `lfv mv` fundamentally cannot: **merging another file's history**. The typical case is where `<f_src>`'s content is a superset of `<f_dst>`'s, and the user does not need to keep both `file-id`s — they want to splice `f_src`'s entire snapshot chain onto `f_dst` and retire `f_src`. This is a semantic history-consolidation operation, categorically different from a path rename. Conflating the two would make both commands harder to understand.

## Possible Future Extensions

- Simple HTML5 interface with visual branch-tree display for a file.
- Hook system, e.g. auto-snap before saving.
- Cross-repository object pool sharing.
- Bridges to git LFS and NAS vendor APIs.

## Open Questions

The following will be resolved based on practical feedback during development:

1. **Rename auto-detection threshold** (`rename.autodetect`): detect only on exact content-hash match, or allow approximate matching at "similarity ≥ N%"? The latter is significantly more complex; leaning toward exact match first.
2. **`lfv revive` default restore point**: restore from the last `object != null` snapshot before deletion, or require the user to specify `<snap>` explicitly? Leaning toward the former as default, with user override available.
3. **History continuation for a same-name new file**: when `lfv revive` could continue history onto a newly tracked file at the same path, should an additional safety confirmation or `--force` be required to prevent users from accidentally splicing a semantically unrelated new file onto old history?
