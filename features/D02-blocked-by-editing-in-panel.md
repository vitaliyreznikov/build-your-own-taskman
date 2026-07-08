---
id: D02
epic: D — Task relations
title: Blocked-by editing in the note panel
size: M
requires: [D01, B06]
novel: false
---

## What
From a task's note panel, the user can add and remove relations — most visibly
"blocked by" links — without hand-editing `relations.md`. The panel shows the
task's current relations and offers controls to add a new one or drop an existing
one, backed by `relations:add` / `relations:remove`.

## Why it exists
Relations are only useful if they're cheap to maintain while you work. Putting
add/remove where the task is read and edited — the note panel — keeps the graph
current without a context switch.

## Acceptance criteria (EARS)
- The note panel shall display the relations the current task participates in
  (including inverse "blocked by" links derived from D01).
- When the user adds a relation from the panel, the system shall append the
  corresponding row to `relations.md` via `relations:add`.
- When the user removes a relation from the panel, the system shall delete that
  row via `relations:remove`.
- After an add or remove, the panel shall reflect the updated relation set.

## Build notes
- UI is `Relations.tsx` inside the NotePanel (B06); IPC is `relations:add` /
  `relations:remove`, both writing `relations.md` through `relationsFile.ts` (D01).
- "Blocked by" is the inverse of a `blocks` row — render it from existing rows;
  don't store a second, redundant row.
- Pair with the typeahead picker (D03) for choosing the other task; the panel
  should not require the user to type a raw id.
