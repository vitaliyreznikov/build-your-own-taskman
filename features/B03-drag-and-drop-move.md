---
id: B03
epic: B — Kanban board UI
title: Drag-and-drop move between columns
size: M
requires: [B01, A03]
novel: false
---

## What
Drag a task card from one column to another to change its state. Dropping a
card updates the task's `Column` field and its position in the global
`order.md`, then persists both to disk.

## Why it exists
Moving a card between columns is the core direct-manipulation gesture of a
Kanban board — it is how the human declares "this task is now in progress /
blocked / done." Because state is the source of truth (separate from the
transient agent status), a deliberate drag is the right, human-in-the-loop way
to change it.

## Acceptance criteria (EARS)
- When the user drops a card in a different column, the system shall set that
  task's `Column` to the target column.
- When the user drops a card, the system shall update the task's position in
  `order.md` to reflect where it was dropped.
- When a move completes, the system shall persist the new column and order to
  disk (via `tasks:move`) using atomic writes.
- When a move fails to persist, the system shall not leave the board showing a
  position that isn't on disk.

## Build notes
- Wiring: the renderer issues a `tasks:move` IPC call; electron-main applies
  the column change to `tasks.md` (A02) and the reorder to `order.md` (A03).
- `order.md` (A03) is a single global cross-board ordering — a move updates that
  one list, not a per-column list.
- Use atomic writes (write-temp-then-rename) so a crash mid-move can't corrupt
  `tasks.md` / `order.md`.
