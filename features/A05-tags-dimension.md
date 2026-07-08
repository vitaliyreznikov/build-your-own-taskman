---
id: A05
epic: A — Markdown data model
title: Tags dimension (boards.md)
size: S
requires: [A02]
novel: false
---

## What
A file, `boards.md`, defines a tag dimension for tasks (e.g. `job`, `personal`).
Each task can carry one or more of these tags; they surface in the UI as the
"Board" column on a card and as a "Tags" filter over the board. This is a
labeling/grouping axis orthogonal to the workflow columns (A04).

## Why it exists
A task has both a *state* (which column) and a *kind* (which area of life/work).
Conflating them would overload the columns; keeping tags as their own flat-file
dimension lets the human slice the same global pile of tasks different ways —
filter to just `job`, or just `personal` — without moving anything between
columns or boards.

## Acceptance criteria (EARS)
- The system shall read the set of available tags from `boards.md`.
- When a task carries a tag, the system shall show it in the card's "Board"
  column.
- When the user selects a tag in the "Tags" filter, the system shall show only
  tasks carrying that tag.
- A task shall be able to carry more than one tag, and tagging shall not change
  the task's workflow column or its global order.

## Build notes
- Keep this dimension distinct from `columns.md` (A04, workflow state) and from
  the later multi-board/`views` epic (which is about board *membership* surfaces).
  `boards.md` here is the flat tag vocabulary; the per-task tag value rides in
  `tasks.md`'s `board` field (A02).
- Historic naming is loose ("Board" column in the UI, "Tags" as the filter, file
  is `boards.md`) — honor the existing names rather than renaming the file.
- Filtering by tag is a view concern; it must not mutate `tasks.md` or `order.md`.
