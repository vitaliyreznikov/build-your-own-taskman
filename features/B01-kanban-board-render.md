---
id: B01
epic: B — Kanban board UI
title: Kanban board render
size: M
requires: [A02, A04]
novel: false
---

## What
The main screen: a Kanban board that reads the `tasks.md` index and the
`columns.md` column definitions and renders one column per allowed state
(NEXT / IN PROGRESS / BLOCKED / DONE / CLOSED), each filled with the task
cards that belong to it. This is the primary surface of the app — everything
else in this epic hangs off it.

## Why it exists
The board is how a human oversees many tasks at a glance. The whole product is
an attention-routing machine, and the board is the top-level view where that
routing happens — you look at columns of cards and see what is where. It is a
pure *view* over the flat files: no state lives in the board, it just projects
`tasks.md` through `columns.md`.

## Acceptance criteria (EARS)
- When the app opens, the system shall render one column for each state defined
  in `columns.md`, in the order they are declared.
- When rendering a column, the system shall show every task whose `Column`
  field in `tasks.md` matches that column.
- When `tasks.md` or `columns.md` changes on disk, the system shall re-render
  the board from the new file contents.
- The system shall place a task with an unknown or missing column somewhere
  deterministic rather than dropping it silently.

## Build notes
- Components: `Board.tsx` owns the layout; `Column.tsx` renders a single
  column and its stack of cards. Cards themselves are B02.
- The board is a projection of the data model — read `tasks.md` (A02) for the
  cards and `columns.md` (A04) for the column set and order. Do not introduce a
  separate board data structure.
- Ordering of cards within/across columns comes from `order.md` (A03) once
  drag-and-drop (B03) is in; for a first cut, a stable read order is fine.
- Which tasks are eligible to show at all is decided by the visibility rules
  (`visibility.ts`) — keep that logic separate from column bucketing.
