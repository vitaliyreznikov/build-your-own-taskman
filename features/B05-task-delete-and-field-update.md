---
id: B05
epic: B — Kanban board UI
title: Task delete and field update
size: S
requires: [A02]
novel: false
---

## What
Editing a task's structured fields — priority, project, column, NextAction —
and deleting a task outright. Both operate on the index (`tasks.md`) through
IPC (`tasks:update`, `tasks:delete`).

## Why it exists
The structured fields live in the index, not the note, precisely so they can be
edited as data without touching the human/AI document. Delete and field-update
are the basic lifecycle operations that keep the board honest — priorities
change, projects get reassigned, and finished-with tasks go away.

## Acceptance criteria (EARS)
- When the user changes a task's priority, project, column, or NextAction, the
  system shall persist that change to `tasks.md` (via `tasks:update`).
- When the user deletes a task, the system shall remove it (via `tasks:delete`)
  and stop showing it on the board.
- When a field is updated or a task deleted, the system shall write the change
  to disk atomically.

## Build notes
- Wiring: `tasks:update` and `tasks:delete` IPC handlers in electron-main
  mutate `tasks.md` (A02).
- Structured fields (priority/project/column/NextAction) live in the index by
  design — updates edit the row, not the `I<N>.md` note body.
- Use atomic writes so an interrupted update/delete can't corrupt `tasks.md`.
- Column changes also happen via drag (B03); both routes converge on the same
  `Column` field.
