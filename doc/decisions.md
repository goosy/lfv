# LFV — Key Design Decisions

This document records **why** LFV is designed the way it is. Each entry explains the rationale behind a specific design choice, including accepted trade-offs and rejected alternatives.

## 1. File-centric design

The file-centric principle stems from these real-world usage scenarios:

- **NAS file sync systems**: each backed-up file has its own history, but cross-file atomic commits are unnecessary.
- **One-note-per-file repositories**: each note evolves independently; changes to note A should not "pollute" note B's history.
- **Design drafts / contracts / single-document archives**: each document tracked individually.
- **Configuration file directories**: each config file evolves independently, without affecting others.

Only allowing snapshots of the whole repository while disallowing snapshots of individual files amounts to requiring cross-file coupling — which is exactly the difference between Git and LFV.

That is, every LFV snapshot is fundamentally an archive of a single file; the repository snapshot command is merely an aggregate of the snapshots of all tracked files in the repository.

LFV treats repository snapshots as optional for the user, while per-file snapshots are mandatory.

## 2. Rewind never destroys history

Snapshots are append-only and always reachable. Any "go back in time" operation is implemented by **creating a new branch**, never making snapshots before HEAD unreachable.

Destructive undo is a constant source of user error and data loss. Turning every rewind into branch creation preserves history by construction. The cost (slightly more complex branch management) is negligible compared with the safety guarantee.

`snapshots.log` is never rewritten. This naturally supports backup (incremental copying always works), auditing (a verifier can replay from the beginning), and crash recovery (partial writes at the tail can be detected and trimmed without corrupting earlier entries).

In-place mutation provides no meaningful benefit while sacrificing all three properties.

## 3. Content addressing + deduplication

LFV follows the same storage mechanism as Git: objects are content-addressed, keyed by the blake3 hash of their raw bytes, so identical content shares one object. A file's persistent identity, by contrast, is its `file-id` — a ULID assigned at track time, independent of both content and path, and its only stable reference.

A `file-id` is assigned when a file is tracked and persists throughout the file's entire lifecycle (including after deletion). It remains the only stable means of reference after a file disappears or its path changes. After a delete or rename the old path still resolves to the file, but once a new file takes that path it points at the new file instead, so a path is not a stable handle. Users and scripts need a stable handle for querying history (`lfv log`), restoring content (`lfv revive`), or continuing history (`lfv relink`).

Paths are convenience aliases for the common case. Any command that accepts `<file>` also accepts either a path or an `file:*` identifier. Both forms are accepted everywhere, so users are never forced to look up a `file-id` during routine work.

## 4. Tracking policy

Files inside the working directory should normally always be tracked. LFV performs an incremental lazy scan before relevant commands: newly created files are automatically tracked; tracked files deleted by the OS are marked as `modified` and shown as `D`; the next `lfv snap` appends a Snapshot according to the current filesystem state.

The LFV mental model is "I have a directory, and every file in it has version history"; the track-everything-by-default decision exists mainly because this model fits most scenarios. Explicit per-file opt-in reverses the model: the user must remember to run `lfv track` after creating every file, and forgetting means losing history. Track-by-default matches expected behavior and is consistent with how backup tools work. Files that need to be excluded are handled through `.lfvignore` (static exclusion list) or `lfv untrack` (dynamic per-file opt-out).

Whether a file stays out of tracking is normally decided by `.lfvignore` and `config.yaml`, but the two have different meanings:

- Paths matched by `.lfvignore` do not exist at all as far as LFV is concerned — they never enter the status table, are not scanned, and never appear in any output. This matches `.gitignore` semantics and keeps the user's cognitive load low.
- The untracked list in `config.yaml` records the user's runtime management decisions; adding or removing tracking there always produces explicit output.

The tracking state in the database status table is determined by these two sources of truth. The status table normally reflects only files that currently exist on disk (with `D` as the exception), which avoids large numbers of "ghost rows" and keeps `lfv status` output an accurate reflection of the real working tree.

The two kinds of non-tracking are distinguished mainly because their runtime output requirements differ.

## 5. Snapshot

A Snapshot is a version record, roughly corresponding to a Git commit.

Snapshot ids use ULID for readability and chronological ordering; tamper resistance is delegated to the independent `digest` field, verified by `lfv verify` (see `design.md §3.3`).

Content-addressed ids (such as Git's SHA hashes) couple identity with tamper resistance, forcing a trade-off between an opaque hash (poor experience) and a readable string (unusable as an integrity proof). LFV separates the two concerns: the ULID id is easy to paste, time-ordered, and content-independent, while the `digest` field provides independent tamper detection. `lfv verify` can validate every Snapshot without touching the id.

Within a snapshot the ULID is fully decoupled from the path; rename/move is recorded as an **immutable event**. Making the path a field of every Snapshot turns the Snapshot chain into the single complete source of truth for both file location and content. This rules out the fragmented source of truth that a mutable `aliases` list would cause.

Without ULIDs, every rename either breaks historical continuity or requires a mutable alias table, which fragments the source of truth.

## 6. Deletion is just a special Snapshot value; history is never lost

`lfv delete` removes the file from disk and marks it as `modified` in the status table. During `lfv snap`, LFV checks the filesystem state of the target file; if the file does not exist, it appends a Snapshot with `object = null` and removes the file's record from the status table. All historical Snapshots remain intact and can be restored at any time with `lfv revive`.

Deletion in LFV is a **state transition**, not erasure. Erasing history would violate the append-only guarantee (Decision 2) and leave `lfv revive` with no way to work. Treating deletion as a Snapshot whose object is null keeps it structurally consistent with the rest of the model — the storage layer needs no special handling.

The two-step design (mark `modified` → confirm at `lfv snap`) gives the user a correction window: while the file is still in `D` state, `lfv mv` can reclassify the event as a rename, avoiding an accidental deletion.

## 7. Symmetry Between Tree Snapshots and File Snapshots

`lfv snap --tree` creates a **Tree Snapshot** in exactly the same way that `lfv snap` creates a File Snapshot: by appending an immutable event record to an append-only log. It is addressed by a ULID, has a single parent pointer, contains a `digest` field for tamper detection, and points to a Tree Object.

A Tree Object is a content-addressed YAML manifest — a path-sorted list of `{path: file-object-hash}` entries covering all tracked files whose current HEAD holds a valid (non-null) object. Tree Objects and File Objects are stored in the same `objects/` directory and deduplicated by content. Two working-directory states with identical content produce the same Tree Object hash, and `objects/` holds only one physical entry.

Tree Snapshots are stored in `.lfv/trees/snapshots.log`, have their own tag set, and are managed through the `--tree` variants of existing commands (`lfv log --tree`, etc.). A repository may contain no Tree Snapshot at all; the tree plane is entirely optional.

`lfv snap --tree` requires that every modified tracked file has already been snapshotted (run `lfv snap` first). This guarantees that the Tree Object is built from the current HEAD of every file and prevents silently producing a stale tree snapshot.

**Why not silently snapshot files inside `lfv snap --tree`**: keeping file snapshots and tree snapshots separate keeps user intent explicit. A Tree Snapshot is a deliberately marked milestone; triggering snapshots of possibly many files as a side effect would bury each file's independent history under unintended snapshot entries. The `--snap-all` flag is the user **explicitly** asking to "snap every modified file one by one with the same message first, then create the tree snapshot" — it merely saves the manual per-file invocations; each file still gets its own independent, visible snapshot in its history, so it is not a silent side effect.

## 8. Single-branch object uniqueness invariant and cycle prevention

**Object uniqueness**: on any single branch of a file's history, the same file-object-hash may form only **one contiguous run of snapshots**; it must not reappear non-contiguously. Contiguous snapshots sharing the same object (a pure rename `R`, a path changed back, and so on) collapse into a single node at the content layer and do not constitute a cycle; what is forbidden is returning to an earlier object.

Without this constraint, a branch could revisit a content state that appeared earlier, producing a backward edge (a cycle) in the content-layer graph. Cycles make content-layer rendering ambiguous: the same node appears twice on the timeline, and its annotations (message, timestamp) become unclear.

**Mechanism**: when `lfv snap` computes the new content hash, it first skips the contiguous ancestor run that shares the new hash (the same run); if an earlier ancestor still has the same object hash, it refuses to append the snapshot and outputs a hint instead:

```
warning: content of docs/note.md matches ancestor snap:2 on branch B
suggestion: run `lfv rewind docs/note.md snap:2` to re-anchor
            this will preserve the intermediate history as branch detour/<anchor-short>/1
```

After the user runs `lfv rewind`:
1. A new branch `detour/<anchor-short>/<n>` is created pointing at the original HEAD (naming rules in spec §4.5) — the intermediate history is preserved in full, and the branch name itself is left untouched.
2. A new snapshot is created on the current branch whose parent points at **the parent of the first snapshot of the run containing the matching ancestor** (not at the ancestor itself), with the new content as its object, so that the new snapshot and the ancestor are two distinct nodes in the Snapshot chain even though their object hashes are identical.

The same constraint also determines two derived rules: `lfv snap` offers no `--allow-empty` (a snapshot with the same object and path as its parent would be indistinguishable from it — see below); and `lfv revive` is implemented as a rewind rather than by appending a "revival" snapshot — appending the pre-deletion content after the deletion event would be exactly the forbidden cycle back to an earlier object.

**Why point to the ancestor's parent, not the ancestor itself**: if the new snapshot's parent were the ancestor, both would share the same object hash and the same parent — they would be indistinguishable nodes in the Snapshot chain, and the rendering could not express a meaningful edge between them.

**Why warn instead of auto-rewinding**: the intermediate history (the detour) may be intentional and worth reviewing before archiving. Surfacing the decision to the user is consistent with LFV's general philosophy of not silently restructuring history.

**`lfv verify` enforcement**: the single-branch uniqueness invariant is a checkable property. `lfv verify` traverses each branch and reports any violation as a data integrity error, not merely a warning. A second occurrence of the same object hash on one branch means the cycle-prevention flow was bypassed, which should not happen under normal operation.

## 9. Topology View Design

Every Snapshot has exactly one parent pointer, forming a directed forest — both for File Snapshot chains and Tree Snapshot chains. Once branches diverge, they evolve independently, and there is no topological merge point (i.e., there is no physical Merge Snapshot with multiple parents). This differs fundamentally from Git's DAG model.

### 9.1 Composite View Structure

The Snapshot directed-tree topology and "content merging" are decoupled: insisting on a directed-tree Snapshot topology does not imply rejecting content-level merging or alignment for single files.

Therefore, in the user-facing view, both Snapshots and Objects are exposed to the user. The user sees Objects as visible nodes, and traces inheritance relationships between them via Snapshot links — the resulting topology in the user view is a DAG.

- **Snapshot layer (storage layer):** Always a directed tree. Every Snapshot has exactly one parent pointer. This layer represents physical storage content and its structure never changes.
- **Content layer (derived layer):** Nodes are object hashes, edges are projected from Snapshot parent relationships onto object identities. Without additional constraints, cycles could appear at this layer (a branch revisiting an object hash previously seen). Decision 8 eliminates such cycles through the "single-branch object-hash uniqueness" invariant, making this layer a DAG.

### 9.2 Design Rationale

**Decoupling the Snapshot layer from the content layer is the prerequisite for a directed-tree topology to hold.** This maps to project goal 3:

> Every branch of a file must be a single-directional **non-branchable path**, without mid-path divergence or merge.

The Snapshot chain is an event log (metadata), and the File Object is content storage (bytes) — they are loosely coupled via the `object` field, rather than having Snapshots directly serve as content nodes. If we use Snapshots as the alignment unit for "content merging," any merge operation would inevitably require two Snapshot chains to converge at some point — i.e., produce a multi-parent Merge Snapshot. This directly violates the directed-tree constraint and dramatically raises implementation costs (requiring handling of multi-parent topology, non-unique historical paths, etc.).

By pushing content-alignment semantics down to the Object layer, both Snapshot chains independently maintain single-parent directed trees without interfering with each other. LFV treats "content merging" and "historical-topology merging" as two separate concerns: when users merge files, what they truly care about is whether content has been unified, not whether multiple historical branches converge at a single Snapshot node. The criterion for "merge complete" is that both chain HEADs point to the same File Object hash — content is unified, and no structural change is needed at the Snapshot layer. The content layer (Object DAG) handles alignment semantics; the Snapshot layer (directed tree) records events only, with clear separation of duties.

On this basis, LFV physically rejects Git-style multi-parent DAG topology in order to preserve **ultimate simplicity and absolute determinism in historical tracing** at the base level:

- **Minimal storage and indexing**: `snapshots.log` needs only an optional `parent` field; snapshot history is naturally physical divergence of singly linked chains.
- **Minimal algorithms**: Topological traversal degenerates into simple O(N) linear backtracking, completely avoiding Git's complex graph topological sorting (Topology Sort) and cycle-detection algorithms.
- **Absolutely pure and readable history:** A directed tree guarantees that the ancestor path of any Snapshot is uniquely determined.
- **No dual-mainline ambiguity:** After a Git merge, branches exist on two paths in the topology with no way to distinguish which is the true mainline. LFV's single-parent topology eliminates this issue at the root while allowing different branches to evolve at different granularities.

### 9.3 Pros and Cons Comparison (Directed-Tree Topology vs DAG)

| Dimension | Git DAG Topology | LFV Directed-Tree Topology |
| :--- | :--- | :--- |
| **Topology Definition** | Each node (Commit) can have one or more parents (multi-parent used to record merges). | Each node (Snapshot) can have at most one parent (parent pointer is unique). |
| **Underlying Complexity** | Extremely high. Requires handling complex cross-path relationships and multi-path reachability. | Extremely low. Data structure is a singly-linked forked tree (directed forest); storage, tracing, GC are all very lightweight. |
| **Integration Mechanism** | **Topology-level Merge (Merge Commit)**: convergence established on the physical graph via multi-parent pointers. | **Reconstruction / Content-level Integration (Rebase/Content Merge)**: no multi-parent integration node in physical form; relies on parent pointer redirection or single-parent Snapshot appending content. |
| **History Readability** | Easily polluted. The single-file history line (`git log <file>`) is blurred by numerous unrelated commits and crossing lines. | Absolutely pure. Ancestor path uniquely determined, perfectly restoring the linear evolution of that single file over time. |
| **Target Scenario** | Multi-person collaboration, multi-file high-frequency merge distributed large software projects. | Single-person, single-file, offline, local — such as notes, configuration files, NAS backups and other single-file history tracking. |

## 10. Complete Deletion of Inappropriate Content

LFV's append-only guarantee means that snapshots within **the traversable range** cannot be modified. Unreachable (dangling) snapshots are exempt from this constraint and can be safely physically deleted.

Based on this, the deletion flow for inappropriate content (e.g., historical versions involving privacy or security issues) is as follows:

1. **Reconstruct history**: `rewind` to the parent snapshot of the one containing inappropriate content, create a new branch. On the new branch, individually `merge --pick` subsequent snapshots from the original branch, editing to remove inappropriate content as needed, forming a new append-only snapshot chain. This entire process operates within existing primitives without violating append-only.
2. **Delete the original branch**: Delete the original branch containing inappropriate content. The snapshot chain on the original branch loses all reachable entry points and enters a dangling state.
3. **Physical purge**: Execute `lfv gc --purge` to delete all dangling snapshots and their associated File Objects.

For inappropriate content appearing across multiple branches, apply steps 1–2 to each relevant branch separately, then execute a single unified `gc --purge`. `gc --purge` is an explicit heavy operation that warns users this is irreversible physical deletion.

By default, dangling snapshots and File Objects are retained — users can still recover them.

**Why Not Append a "Set to Null" Record**?

An intuitive approach would be to **append a new Snapshot (with `object = null`)** on the snapshot chain containing the inappropriate content. However, this approach has the following issues:

- **Traversal cost grows linearly with branch count**: All historical snapshots that reference the File Object must trace that post-append deletion record during rendering — each display adds one more lookup.
- **Semantic confusion**: "content disappearance" and "file deletion" would share the same mechanism (`object = null`), complicating view-layer logic.
- **Tree-plane decoupling gap**: Tree Snapshots reference File Object hashes via Tree Objects. Appending a set-to-null record does not modify the Tree Object's manifest — the tree snapshot would still display the deleted content, leaving users with inconsistent history.

LFV therefore chooses to handle elimination in **one-time batch processing during reconstruction**, rather than dynamic tracing at query time. The view-layer cost is concentrated in a one-time operation; subsequent normal queries have zero overhead.

## 11. Tree-plane design principles

### 11.1 tree-id uses content hash, not ULID

The tree-id (the identity of a Tree Object) is the content hash `blake3(canonical manifest bytes)`, not an independent ULID like a file-id.

A repository logically has only "one tree" — the set of all tracked files in the working directory, which changes as content changes. It does not need a stable identity of "which tree this is". Using the content hash as tree-id provides natural deduplication (two identical workspace states share the same Tree Object) and avoids the overhead of maintaining a separate identity (ULID + meta.yaml) for trees. This differs from file-id design — files need a stable identity across renames and moves, hence a ULID decoupled from paths; trees do not need identity across content changes, so a content hash suffices.

### 11.2 Tree has no branches, only tags and HEAD

Tree Snapshot chains have no branch set. They have only: tags (prefixed with `tree:`, globally unique), and a single HEAD pointer (the current Tree Snapshot).

The core value of branches is to support "multiple independent evolution lines of the same file" — a typical need on the file plane (users need to experiment with different content on different branches). Trees are milestone-style repository snapshots; their usage pattern is linear progression, and they neither need nor should have multiple parallel evolution lines. Introducing tree branches would only increase cognitive load without meaningful benefit.

**Tree HEAD is stored in a separate file `.lfv/trees/HEAD`**: tree HEAD is a current state that cannot be reconstructed from the Snapshot chain (it records "which node the user is currently on in tree history", not which Snapshot is the latest). Like a file HEAD and the branch tables, it records user intent: the branch-pointer cache in `index.db` is rebuilt from `branches.yaml` during `rebuild-index`, whereas a lost tree HEAD cannot be derived from anything, so it is persisted in a file of its own as its own source of truth, outside the rebuildable `index.db`, where `rebuild-index` can never overwrite it.

`lfv switch <branch>` only operates on file branches; there is no tree switch operation (because trees have no branches). Tree position changes are done only via `lfv rewind <TS-ish>`.

### 11.3 tree rewind does not change tracking identity, and does not touch config

`lfv rewind <TS-ish>` does not directly call the `track`/`untrack`/`delete`/`revive` commands, does not modify the dynamic untracked list in `config.yaml`, and does not change the tracking identity of any file-id — it allocates no new file-id, retires none, and neither registers nor deregisters tracking. It updates `.lfv/trees/HEAD` and performs byte-level operations (write or delete) on working directory files; for a file in the manifest whose file-id is still active, it additionally moves that file's branch pointer and refreshes the cached fields of its `tracked` row through `rewind_file` / `switch` (see §11.4) — that is what rewinding the file's own content requires, and is a different matter from tracking identity. For files not in the manifest (the working-tree file is merely deleted) and for entries whose file-id is no longer active (the bytes are merely written back), it is a pure byte operation.

`config.yaml` and the life and death of a file-id belong to the responsibility boundaries of the `track`/`untrack`/`delete`/`revive` commands. If tree rewind crossed these boundaries, users would find it hard to predict which commands have side effects on config, breaking command responsibility clarity.

After byte operations, the existing lazy scanning mechanism (`design.md §4.3`) will naturally detect changes and drive state updates on the next `lfv status` or `lfv snap`. "Newly created files" and "disappeared files" produced by tree rewind are identical in effect to files directly manipulated by the OS, so the user's mental model does not need special casing.

### 11.4 tree rewind uses Fast-Forward preference per file

`lfv rewind <TS-ish>` does not directly execute rewind (which would create a new branch) for each file that needs restoration. Instead, it first checks whether a Fast-Forward path exists:

- **FF path**: If the current HEAD of a branch has an object hash equal to the target hash, directly `switch` to that branch without creating a new branch. Prefer the current branch (no switching cost); if the current branch does not match, pick the most recently created matching branch.
- **rewind path**: Otherwise, perform a standard rewind, automatically creating a new branch (`rewind/<anchor-short>/<n>`).

Tree rewind is a batch operation that may affect dozens of files. If every file unconditionally created a new branch, the resulting branch noise would greatly interfere with users' reading of each file's history. The FF path covers most tree rewind scenarios in practice (because tree snapshots are typically created immediately after each file's snap, at which point each file HEAD's object is exactly the object recorded in the tree object), making zero branch pollution the norm. Only when truly rewinding to a non-HEAD position does a new branch get created, consistent with Decision 2 (Rewind never destroys history).

This is isomorphic to Git's Fast-Forward merge: no extra node is produced when FF is possible; branching happens only when necessary.

### 11.5 Tree Object uses YAML block sequence, not JSON

The Tree Object manifest uses YAML block sequence format (`- "path": hash`), not a JSON array (`[{"path":...,"object":...}]`).

**Rejected alternative**: JSON array. Canonicalization of JSON requires additional conventions (no extra whitespace, fixed key order, no trailing commas). Implementations must use a controlled serializer rather than plain `to_string()`, separating canonicalization requirements from the format specification, which easily leads to inconsistent hashes due to implementation omissions.

**Reasons for choosing YAML block sequence**:

**Storage format = canonicalization format**: Each line `- "path": hash` is fully deterministic (quotes, sorting, single newline). `blake3` digests the byte string directly, with no extra canonicalization step. The canonicalization constraint is embodied in the write rules, not in an additional serialization protocol.

**Safety of paths as keys**: In YAML, bare keys have special meanings for characters like `#`, `[`, `{`, `,`, `&`, `*`. The path character set conflicts with these. Double-quoted keys completely eliminate all special-character issues — including paths with spaces (very common on Windows/NAS scenarios). Paths are always stored with Unix separator `/`, not `\`. The only character that needs escaping inside the key is `"`, which almost never appears in real paths, so the overhead of quotes is negligible.

**External compatibility**: This format is a valid YAML subset; external programs can read it with a standard YAML parser. LFV internally can also parse line by line with a simple regex `/^- "(.+)": (\S+)$/`, without relying on a full parser. JSON also has external compatibility, but YAML is slightly better in character saving and parsing simplicity.

**Character efficiency**: Each entry saves about 20% of characters (~35 characters per YAML line vs. ~45 per JSON object). Tree Objects are usually below the `min_bytes` compression floor, stored as `.raw`, so character efficiency directly translates to storage efficiency.

## 12. Two storage object categories, each with two subtypes

The truly immutable storage objects in the repository fall into two categories — **Object** and **Snapshot** — each with two subtypes. Everything else falls into two classes: the rebuildable cache inside `index.db` (see spec §3.4 and `design.md §3.6`), and the mutable files that record user intent and cannot be derived from the snapshot chain — `HEAD`, the branch table, the tag table, and `config.yaml` (see `design.md §3.7` and §3.8).

| Category | Subtype | Contents | Addressing |
| --- | --- | --- | --- |
| Object | **File Object** | Raw file bytes (compressed) | `blake3(raw bytes)` |
| Object | **Tree Object** | Path-sorted `{path: file-object-hash}` manifest (YAML) | `blake3(canonical manifest bytes)` |
| Snapshot | **File Snapshot** | Event record for one file | `snap:<ULID>` |
| Snapshot | **Tree Snapshot** | Event record for the working directory as a whole | `snap:<ULID>` |

Both Object subtypes are stored in the same `objects/` directory, content-addressed and fully immutable. Both Snapshot subtypes are append-only and follow the same format (JSON Lines), but live in separate directories:
- File Snapshot: `.lfv/files/<ULID>/snapshots.log`
- Tree Snapshot: `.lfv/trees/snapshots.log`

**Why a Tree Object is needed now**: LFV's use cases include milestone snapshots of the whole working directory (e.g. "a complete edition of this book"). A Tree Object makes this a first-class operation while preserving the core invariant — content identity is determined by hash, not by snapshot id. Two working-directory states with identical content produce the same Tree Object hash and share one entry in `objects/`, exactly like the mechanism by which two files with identical content share one File Object.

**Why Tree Object stores file-object-hashes, not file-snapshot-ids**: The node identity in LFV's history graph is content (object hash), not the metadata record (snapshot id). A Tree Object that references file-object-hashes is content-addressed end-to-end; referencing snapshot ids would couple the tree's identity to incidental metadata (message, timestamp), breaking deduplication and the content-node semantics.

## 13. Dual compression thresholds

`min_bytes` (floor, default 4 KiB) and `max_bytes` (ceiling, default 16 MiB) handle files that are too small and too large respectively. `blake3` is always computed from raw bytes. Objects that skip compression, or for which compression is ineffective, are stored as `.raw`; compressed objects are stored as `.zstd` (see `design.md §3.2`).

A single threshold cannot handle both edge cases. Very small files may grow after compression due to framing overhead. Very large files can create memory spikes during compression (the whole file must be read into memory). Dual thresholds explicitly distinguish both cases, while `reject_if_larger` handles the fallback for already-compressed binaries (e.g. images) that offer no compression benefit. `blake3` is always computed from raw bytes, ensuring the hash is independent of the storage format.

## 14. SQLite as the index

Metadata requiring random updates — the status table, branches, tags, the global fast index over files, and so on — lives in `index.db`.

It serves only as a cache; source-of-truth data lives in files.

The status table and branch pointers are **mutable** and require efficient random-access reads and atomic updates. A pure-file approach (such as one YAML file per file) is workable at small scale, but performance degrades sharply with thousands of tracked files. SQLite provides ACID transactions, efficient indexed lookups, and a single-file deployment model — no daemon, no network. The historical Snapshot chain, by contrast, is append-only and accessed sequentially, so a plain JSON Lines file is simpler and entirely sufficient.

## 15. `lfv mv` and `lfv relink` are separate commands

`lfv mv <src> <dst>` performs path operations only; `<dst>` may not be a `file-id`. `lfv relink <f_src> --onto <f_dst>` is dedicated to history continuation.

If `lfv mv <src> <f_dst-id>` were allowed to mix path manipulation with history continuation, users would naturally assume that "the `file-id` left behind afterwards is the `<dst>` one" — whereas the identity that actually survives the continuation is the `--onto` side (i.e. `f_dst`), and `f_src` is retired. This mental-model confusion would almost inevitably lead to mistakes.

`lfv relink` also solves a second problem that `lfv mv` fundamentally cannot: **merging another file's history**. A typical case is when the content of `<f_src>` is a superset of `<f_dst>` and the user does not need to keep both `file-id`s, wanting instead to splice the full snapshot chain of `f_src` onto `f_dst` and then retire `f_src`. This is semantically a history-consolidation operation, a different level of concept from a plain path rename; forcing them together would only make both commands hard to understand.

## 16. Tree reverse references use `tree_file_refs` DB table

Reverse references for which Tree Snapshots contain each file are stored in the `tree_file_refs` table in `index.db`, not in a separate file under each file-id directory. Fields: `tree_snap_id` (`snap:<ULID>`), `file_id`, `file_object` (blake3 hash). The source of truth is `.lfv/trees/snapshots.log` + `.lfv/objects/`; `rebuild-index` reconstructs the table (the manifest contains no file-id, so during a rebuild each entry is attributed to "the latest file snapshot matching `(path, object)` that predates the tree snapshot's timestamp" — see `design.md §3.10`).

**Rejected alternative**: Maintain a YAML file under each file-id directory (`.lfv/trees/.yaml`) with keys = `snap:<ULID>` and values = file-object-hash. This approach produces many small file writes when many files are involved, and lacks transaction guarantees (a crash during writing can cause inconsistency).

**Reasons for using the DB table**: `index.db` provides ACID transactions, efficient single-table queries, and the `tree_file_refs` table is purely a cache (the source of truth is in `.lfv/trees/snapshots.log` + `objects/`). Putting it in the db keeps semantics consistent with other rebuildable indexes; `rebuild-index` rebuilds it uniformly, without risk of missing reverse references for some file.

**Why snap_id as primary key (not tag name)**: Every Tree Snapshot has a message, and is meaningful even without a tag. Using snap_id as the primary key ensures that all Tree Snapshots have reverse references; when rendering, the tag is looked up dynamically from `.lfv/trees/snapshots.log`: if a tag exists, display the tag name; otherwise display the message. Both are meaningful. "Whether a tag is applied" only affects display, not the completeness of history.

## 17. Namespaces and reference syntax

Every special reference on the CLI uses a multi-letter namespace prefix that can never overlap with a path:

| Prefix | Meaning |
| ---- | ---- |
| `file:<ULID>` | file-id |
| `snap:<ULID>` | snap-id (shared by the file and tree planes, globally unique) |
| `tree:<tag>` | tree tag |
| `work:<path>` | current working-area content (a special unsaved File Object) |
| `blake3:<hex>` | object hash / tree-id |

A file reference at a historical version is written `<ref>:<file>` (`<ref>` being a branch or tag name), matching git's `<rev>:<path>`. Branch and tag names may not contain `:` and may not be a namespace name (`file`, `snap`, `tree`, `work`, `blake3`); file names are not restricted by this.

**Why**:

- The earlier `f_<ULID>`, `snap_<ULID>` and `f_/<path>` prefixes overlapped with valid file names (e.g. `snap_notes.md`); parsing needed extra conditions such as "is the rest a ULID" and still had gaps. With namespaces ending in `:`, and trackable paths containing no `:` (§20), a prefix and a path can never overlap syntactically, so argument parsing is purely syntactic and needs no index lookup.
- Prefixes are multi-letter: a single-letter prefix (such as the earlier `t:`) clashes with Windows drive-letter syntax (`t:foo` is a relative path on drive T).
- `blake3:` is kept rather than renamed to `object:`: carrying the algorithm name keeps old data identifiable if the hash algorithm ever changes, and the on-disk format matches the CLI spelling.
- `work:` is separate from `file:`: the working-area literal means the working area's content at this moment, a file-id means the file itself; different meanings get different prefixes.
- `tree:<snap-id>` is dropped: snap-ids are globally unique and the index tells which plane one belongs to, so a `<TS-ish>` / `<TO-ish>` position takes the snap-id directly.
- Directory names under `files/` are the bare ULID: Windows file names cannot contain `:`, so the text form `file:<ULID>` cannot be a directory name.
- `<file>@<ref>` was rejected: `@` is a valid file-name character (e.g. `icon@2x.png`), so parsing had to look the whole token up as a path first and split only on a miss, and when both readings resolved it could only report an ambiguity. `<ref>:<file>` splits at `:` and is unambiguous at the syntax level.
- Branch and tag names may not be namespace names, otherwise `tree:v1.0` could be either a tree tag or "file `v1.0` on branch `tree`".

## 18. Small-tool philosophy

CLI subcommands should remain clear, composable, and scriptable; LFV avoids premature abstraction toward graphical interfaces or services.

LFV is designed for automation, shell scripts, and integration with external tools (sync scripts, editors, CI). Keeping clean CLI conventions a priority is what makes the tool composable. A GUI or daemon layer can always be built on top of a stable CLI; the reverse is painful.

## 19. `diffy` for diff and three-way merge

spec §4.7 requires three-way merge with conflict markers, so the deciding factor is not the diff algorithm itself but whether the crate ships a three-way merge of its own.

- **`diffy` (adopted)**: Myers diff, unified patch generation and application, and `merge(base, ours, theirs)` three-way merge (conflict markers in either merge or diff3 style). A single crate covers both the diff and the merge requirement. Its conflict-marker labels are fixed as `ours` / `theirs`, while spec §4.7.3 requires `ours (main)` / `theirs (feature / snap:...)`; the `merge` module performs one line-prefix substitution on the output.
- **`imara-diff` (rejected)**: used by gitoxide and the fastest option, but has no three-way merge.
- **`similar` + hand-written diff3 (rejected)**: the largest amount of work, with benefits only in word-level highlighting, which LFV does not need.

## 20. Trackable paths follow the Windows file-name rules

A trackable path contains no control characters and none of `< > : " \ | ? *`; no path component ends with `.` or a space or is a Windows reserved name (`CON`, `NUL`, `COM1`, ...). The characters Linux and macOS forbid (`/` and NUL) are a subset of Windows', so the rule holds on all three platforms with nothing to add.

**Why**: history must be restorable on any platform (cross-device sync). If LFV records a file name that some platform cannot create, rewind / revive on that platform fails, and the problem only surfaces at that moment. Applying the strictest platform's rules everywhere prevents such records at the source. As a side effect paths contain no `:`, so namespace prefixes (§17) never overlap with paths.

**Handling of invalid paths**: auto-track does not register them and only issues a warning with a hint to add them to `.lfvignore`, and the scan continues; an explicit `lfv track <file>` is an error, because the operation the user explicitly asked for cannot be done.

## 21. How path arguments are written

`a`, `./a`, `../a` are relative to the current working directory; `/a/b`, starting with `/`, is a repository-absolute path rooted at the repository root; operating-system absolute paths and drive letters are never accepted. Every form is normalized to a path relative to the repository root, and one that leaves the repository is an error. `\` in input is read as `/`, but output and hints use `/` only. In MinGW / Git Bash, write `//a/b` to avoid MSYS path rewriting.

**Why**: keeping current-directory semantics matches command-line habits (tab completion yields relative paths); repository-absolute paths let one command be written the same way from any subdirectory on any platform; rejecting drive letters and OS absolute paths makes path syntax platform-independent and removes any need to disambiguate the `:` in a drive letter. `\` can never occur in a trackable path (§20), so accepting it as a separator is unambiguous; it is there only to accommodate PowerShell completion and is not encouraged.

## 22. Platform differences are left to the user; LFV only detects and reports

- **Paths differing only in case**: on a case-insensitive filesystem, if the repository records two active paths that differ only in case, or an operation would write out two such paths, LFV reports an error and tells the user to enable case sensitivity for the directories involved; otherwise it runs normally.
- **Path length**: the Windows path-length limit is for the user to handle; LFV only reports the cause when a filesystem operation fails because a path is too long.

**Why**: both depend on the user's system settings, and LFV cannot make the right choice for the user; renaming or truncating paths automatically would break the premise that a path is a historical fact.

## 23. A `null` snapshot breaks an object run

Under branch object uniqueness (§8), `null` takes no part in uniqueness but does break a run: `O1 → null → O1` is a non-contiguous reappearance of O1.

**Why**: going back to O1 after a delete is done by rewinding / reviving to the snapshot before the delete, not by appending another O1 after the `null`. A `null` snapshot always stays at the end of some branch, and later work continues from its parent. The only case that appends after a `null` is a relink seam; if the seam content equals the pre-delete object, it is refused as a loop and the user is directed to `lfv revive` instead.

## 24. `<file>` may be omitted for `--continue` / `--abort`

Passing `<file>` is recommended. When omitted, LFV looks for the merge / rebase in progress: exactly one file in progress → act on it; several or none → error.

**Why**: `REPLAY.yaml` is stored per file, so several files can be in the middle of a merge / rebase at once. Always requiring `<file>` is clearest but verbose in the single-file case; allowing only one operation per repository is too restrictive. "Omit when unique, error when not" covers both and never acts on the wrong file.

## 25. The same-path former file-id hint is recomputed on every status

When `lfv status` tells a newly tracked file that "this path previously belonged to a deleted file-id", the former file-id is not cached in the `tracked` table; it is looked up in the logs each time.

**Why**: such `A`-state files are rare and the hint disappears after the first snap, so recomputation costs nothing noticeable; a cache column would need a schema change and an extra recomputation rule in `rebuild-index`, which is not worth it.

## 26. The source of `lfv mv` may be a file-id

In `lfv mv <old-file> <new-file>`, `<old-file>` accepts a path or a file-id, `<new-file>` a path only.

**Why**: a file-id always corresponds to a path (its current path or the path in its HEAD snapshot), so naming the source by file-id is unambiguous, and it is more convenient for files already gone from disk such as `D` rows. The target still accepts paths only, for the same reason as before: if the target could be a file-id, users would easily assume the surviving identity is the target's (splicing history is what `lfv relink` is for).

## 27. Paths are NFC; the on-disk normalization form is probed

A `RepoPath` is always recorded and compared in Unicode NFC. To access the disk, each path component is tried as NFC, then NFD, and only if neither exists is the directory listed and compared; no persistent "`RepoPath` → on-disk name" map is kept. Two on-disk names that differ only in normalization form are an error left to the user. Normalization and case-collision detection both live in the working-area access layer of `scan`.

**Why**: the same name has different bytes on macOS (NFD) and on Windows / Linux (usually NFC); without one normal form, a file moved across devices would look like a rename or a new file. A persistent map records the on-disk names of one machine, but `.lfv` is copied across devices and such a map is meaningless on another machine; probing per component needs no persistent state and is cached only within the process. Case-collision detection likewise depends on how the disk actually behaves, so it belongs in the same layer.

## 28. Branch and tag names may not contain control characters

Branch and tag names may not contain control characters; a name that does is refused at creation. HEAD files therefore always hold a single line with a plain branch name and need no escaping.

**Why**: control characters would break the single-line HEAD format and the line-oriented CLI output, could inject escape sequences into the terminal, and are hard to type on a command line. Unlike file names, branch and tag names are always created through LFV, so refusing a bad name does not disturb data the user already has, as restricting OS file names would; with the namespace-keyword restriction already in place, forbidding control characters costs almost nothing more.

## 29. `anchor-short` uses the last 8 characters of the ULID, unlike the file-id abbreviation's first 8

The `anchor-short` in a preserved branch name (`<kind>/<anchor-short>/<n>`, spec §4.5) takes the **last 8 characters** of the preserved snapshot's ULID; the `file-id` display abbreviation in `lfv status` (spec §4.3.1) takes the **first 8 characters**. The difference is deliberate, not a typo.

A ULID is Crockford Base32-encoded: of its 26 characters, roughly the first 10 are a 48-bit timestamp and the remaining 16 are 80 bits of randomness. The file-id abbreviation takes the first 8 characters, landing in the timestamp segment -- files created around the same time get similar-looking file-id prefixes, which suits how a person reads `log`/`status` output in chronological order and types short prefixes by hand; it's an optimization for human display. `anchor-short` takes the last 8, landing in the random segment -- when several rewind / detour / rebase / revive operations happen in quick succession (for example one `lfv rewind <TS-ish>` touching dozens of files at once), a timestamp-segment prefix would look nearly identical across them and rely almost entirely on the trailing `<n>` to disambiguate; the random segment collides far less often, reducing that reliance on `<n>`.

## Possible Future Extensions

- Simple HTML5 interface with visual branch-tree display.
- Hook system (e.g., automatic snapshots before save).
- Cross-repository object pool sharing.
- Bridges to Git LFS and NAS vendor APIs.
