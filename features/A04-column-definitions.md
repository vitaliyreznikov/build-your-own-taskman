---
id: A04
epic: A — Markdown data model
title: Column definitions (columns.md)
size: S
requires: [A02]
novel: false
---

## What
A file, `columns.md`, lists the allowed board columns — the task states — in
display order: `NEXT`, `IN PROGRESS`, `BLOCKED`, `DONE`, `CLOSED`. The `column`
field on each row in `tasks.md` must be one of these; the Kanban board renders one
column per entry, left to right.

## Why it exists
The set of states is itself data, kept in a flat file rather than hard-coded, so
the workflow's shape is legible and adjustable without touching app code. It also
gives every other feature a single authoritative vocabulary: what counts as
"done", what counts as "in progress", which is exactly what the terminal
auto-move and the LLM-status layer later build on.

## Acceptance criteria (EARS)
- The system shall read the ordered list of columns from `columns.md` and render
  the board's columns in that order.
- When a task is assigned a `column` in `tasks.md`, the value shall be one of the
  columns defined in `columns.md`.
- When a task's column is not set, the system shall place it in the default
  starting column (`NEXT`).
- When a task moves to `DONE` or `CLOSED`, downstream features (e.g. notification
  auto-clear) shall be able to treat those as terminal states.

## Build notes
- The canonical set is `NEXT / IN PROGRESS / BLOCKED / DONE / CLOSED`; keep them
  as the source of the board's column list and of validation for `tasks.md`'s
  `column` field (A02).
- `IN PROGRESS` is special: opening a task terminal auto-moves the task here (see
  the terminals epic). Keep the state name stable so that slug/name matching in
  that feature stays reliable.
- Treat `DONE` and `CLOSED` as terminal for the purposes of surfacing/visibility
  and clearing notifications.
- Order in the file is display order on the board — editing the file reorders the
  columns.
