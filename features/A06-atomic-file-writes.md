---
id: A06
epic: A — Markdown data model
title: Atomic file writes
size: S
requires: []
novel: false
---

## What
All writes to the on-disk data files go through a single atomic writer
(`atomicWrite.ts`): write the new content to a temporary file, then rename it over
the target. The rename is atomic on the filesystem, so a reader (the app, an
agent, a git commit) never sees a half-written file, and a crash mid-write cannot
corrupt an existing file.

## Why it exists
The whole data model is flat files that are read and rewritten constantly — by the
app, by agents, by the sync engine. If a write is interrupted partway, the file
that everything depends on (`tasks.md`, `order.md`, a note) could be truncated or
garbled. Write-temp-then-rename makes every write all-or-nothing, which is the
safety floor under a database-free, files-as-source-of-truth design.

## Acceptance criteria (EARS)
- When the system writes any data file, it shall first write the full new content
  to a temporary file and then atomically rename it over the destination.
- When a write is interrupted (crash, power loss) mid-operation, the destination
  file shall retain its previous complete content, never a partial write.
- A concurrent reader shall observe either the old complete file or the new
  complete file, never an intermediate state.
- All data-file writers (index, order, columns, boards, notes, and later JSON
  state files) shall route through the atomic writer.

## Build notes
- Implement once (`atomicWrite.ts`) in the Electron main process and use it
  everywhere; do not let any feature write a data file with a plain truncating
  write.
- Put the temp file on the same filesystem/directory as the target so the rename
  is a true atomic rename, not a cross-device copy.
- This is a foundational primitive from the initial commit — build it before the
  index (A02), ordering (A03), and sync (A09) that all depend on it.
