# LFV — Lightweight File Versioning

**LFV is a timeline for one file.**

It versions each file independently – its own history, branches, tags, and snapshots.  
No multi‑file commits, no repository‑wide state.

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
lfv rewind notes.md snap_01HXYZ   # automatically creates a new branch

lfv mv notes.md archive/          # rename is a first‑class event
lfv delete old.md && lfv snap     # deletion is part of history
lfv revive old.md                 # restore from anywhere

```

## How to Use It (Everyday Commands)

| Command | What it does |
| --- | --- |
| `lfv status` | Show modified / renamed / deleted files |
| `lfv snap [-m "msg"] [file]` | Snapshot one file or all modified |
| `lfv log <file>` | Show history (supports `--branch`, `--graph`) |
| `lfv diff <file> [<snap> [<snap2>]]` | Compare working tree or snapshots |
| `lfv rewind <file> <snap>` | Restore old version – auto‑branches |
| `lfv mv <old> <new>` | Rename/move, recorded in history |
| `lfv delete <file>` | Remove file (snapshot on next `lfv snap`) |
| `lfv revive <file>` | Restore a deleted file |
| `lfv branches <file>` / `lfv switch` | Branch management per file |
| `lfv tag <file> <snap> <name>` | Tag a snapshot |

## Architecture (In One Paragraph)

LFV stores only two immutable object types: **Object** (file content, addressed by `blake3` hash, deduplicated and compressed) and **Snapshot** (event record with path, object pointer, parent, message, timestamp – stored in JSON Lines).

Mutable state (branches, tags, HEAD, working tree status) lives in an `index.db` (SQLite) – it’s a reconstructable cache, not part of history.

No tree objects, no staging area, no multi-file commits. Written in Rust.

## Roadmap

* **v0.1** – `init`, `track`, `snap`, `log`, `status`, `show`
* **v0.2** – `diff`, `rewind`, `branches`, `switch`
* **v0.3** – tags, `export`, `gc`, `verify`
* **v1.0** – stable CLI, complete docs, cross‑platform CI
