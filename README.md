# LFV — Lightweight File Versioning

**LFV is a timeline for one file.**

It versions each file independently – its own history, branches, tags, and snapshots.  
No multi‑file commits; repository‑wide milestones (tree snapshots) are optional.

Use LFV for documents, notes, config files, backups – anything that lives outside a code repo.

## Who LFV Is For

LFV is especially suitable for:
- **NAS users**: Each backed-up file keeps its own independent history.
- **Personal notes / PKM**: Notes evolve independently without polluting each other's timeline.
- **Design drafts & contracts**: Easy diff, rollback, and clean single-document archiving.
- **Configuration management**: Keep configuration file histories isolated per file.
- **Single-file tracking**: Track standalone files without a heavy repository-centric Git workflow.

It is **not** intended for large collaborative development or merge-heavy software engineering project management. For those scenarios, Git remains the right tool.

## Why LFV Exists

Git’s core abstraction is *a commit of an entire working tree*.  
LFV’s core abstraction is *a timeline of one file*.

| Capability | Git | LFV |
|------------|-----|-----|
| History scope | Repository‑wide | Per‑file |
| Commit model | Multi‑file commit | Single‑file snapshot |
| Branches | Repository‑level | File‑level |
| Rename handling | Tree diff inference | Native history event |
| Rewind | `reset` / `checkout` | Auto‑created branch |
| Mental model | Project management | File lifecycle |

LFV behaves like a filesystem with history – not a collaboration platform.

## Quick Start

```bash
cd ~/Documents
lfv init                # creates .lfv/

lfv status              # auto‑discovers files, shows changes
lfv snap -m "first version"

lfv log notes.md
lfv diff notes.md
lfv rewind notes.md snap:01HXYZ   # automatically creates a new branch

lfv mv notes.md archive/notes.md  # rename is a first‑class event
lfv delete old.md && lfv snap     # deletion is part of history
lfv revive old.md                 # restore from anywhere
```

## How to Use It (Everyday Commands)

| Command | What it does |
| --- | --- |
| `lfv status` | Show modified / renamed / deleted files |
| `lfv snap [-m "msg"] [file]` | Snapshot one file or all modified |
| `lfv log <file>` / `lfv log <branch>:<file>` | Show history (supports `--all`, `--graph`, `--limit`) |
| `lfv diff <version> [<version2>]` | Compare a version with the working tree, or two versions |
| `lfv rewind <file> <snap>` | Restore old version – auto‑branches |
| `lfv mv <old> <new-path>` | Rename/move, recorded in history (`<old>` may be a path or a file-id) |
| `lfv delete <file>` | Remove file (snapshot on next `lfv snap`) |
| `lfv revive <file>` | Restore a deleted file |
| `lfv branches <file>` / `lfv switch` | Branch management per file |
| `lfv tag <file> <snap> <name>` | Tag a snapshot |

## Architecture (In One Paragraph)

LFV keeps content in an immutable, content‑addressed object store (**File Objects** and **Tree Objects**, addressed by `blake3` hash, deduplicated and compressed) and records history as append‑only **Snapshots** in JSON Lines, each protected by a digest: File Snapshots per file, and optional Tree Snapshots that mark repository‑wide milestones.

Branches, tags and HEAD are small YAML files kept per file; `index.db` (SQLite) is a rebuildable cache of working‑tree status and lookups, not part of history.

No staging area, no multi‑file commits. Written in Rust.

Snapshot history forms a **directed tree (forest), not a DAG**: each Snapshot has exactly one parent pointer. Branches diverge and stay independent — there is no topological merge point. To bring changes across branches, `lfv merge` / `lfv rebase` replay them as new snapshots, using a content merge anchored on snapshots that share the same file object; existing snapshots are never rewritten.

## Roadmap

* **v0.1** – `init`, `track`, `snap`, `log`, `status`, `show`
* **v0.2** – `diff`, `branches`, `rewind`, `switch`
* **v0.3** – tags, `export` / `import`, `gc`, `verify`
* **v0.4** – `merge` / `rebase`, conflict handling
* **v0.5** – tree plane (`snap --tree`, `log --tree`, `rewind tree:`, `tag --tree`), performance tuning
* **v1.0** – stable CLI, complete docs, cross‑platform CI
* **v1.1** – remote repositories
