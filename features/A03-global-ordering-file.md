---
id: A03
epic: A — Markdown data model
title: Global ordering file (order.md)
size: S
requires: [A02]
novel: false
---

## What
A single file, `order.md`, defines one global, cross-board ordering of all tasks.
It is the ordered list of task ids that the board uses to decide the vertical
sequence of cards — independent of which column or board a task sits in.

## Why it exists
Order is a property the human cares about ("what's near the top of my mind") and
it needs to be stable and explicit, not implied by row position in `tasks.md` or
by a timestamp. Pulling ordering into its own flat file keeps `tasks.md` about
*what a task is* and `order.md` about *where it ranks* — each file stays small,
single-purpose, and easy for an agent to reason about and rewrite.

## Acceptance criteria (EARS)
- When the board renders a column, the system shall order its cards according to
  the sequence in `order.md`.
- When the user drags a task to a new position, the system shall update
  `order.md` to reflect the new global order.
- When a task is created, the system shall insert its id into `order.md`.
- When a task is deleted, the system shall remove its id from `order.md`.
- The ordering shall be global (one sequence across all columns and boards), so a
  task keeps its relative rank regardless of the surface it is shown on.

## Build notes
- Store just the ordered list of ids; the fields themselves live in `tasks.md`
  (A02). `order.md` is a projection, joined by id.
- A single global order (rather than per-column order) keeps the model simple and
  means moving a task between columns (drag-and-drop) doesn't scramble its rank.
- Moving a card is two writes: the `column` in `tasks.md` and the position in
  `order.md`; both go through the atomic writer (A06).
- Reconcile gracefully: an id in `order.md` with no row in `tasks.md` (or vice
  versa) should not crash the board — treat `tasks.md` as the set of truth and
  `order.md` as the ranking hint over it.
