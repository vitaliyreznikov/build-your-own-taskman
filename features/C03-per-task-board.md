---
id: C03
epic: C — Boards / views / filters
title: Per-task board (auto-shows children)
size: M
requires: [C01]
novel: false
---

## What
Any task can have its own board at `views/I<N>.md`. That board automatically
shows the task's child tasks, plus any tasks added to it manually. It gives a
task with subtasks a dedicated workspace without the user having to build a board
by hand.

## Why it exists
When a task fans out into subtasks so several agents can work in parallel, those
subtasks need somewhere to live together. A per-task board is that somewhere: it
materializes the parent → children structure as a Kanban surface, on demand.

> In the author's words: *"boards for any task with subtasks. They should not be
> created at advance but I need button in the task tab."* And, on parallelism:
> *"Feel free to create subtasks in our system so different processes can work in
> parallel."*

## Acceptance criteria (EARS)
- When the user opens the per-task board for task `I<N>`, the system shall
  automatically show all tasks whose parent is `I<N>`.
- The system shall also show any task ids listed manually in `views/I<N>.md`.
- The system shall not create a per-task board in advance; it comes into being
  when requested for that task.
- The system shall read child relationships and task state from the global index,
  so the per-task board reflects live state (same as any other board).

## Build notes
- Membership = (tasks whose `parent` == `I<N>` in `tasks.md`) ∪ (ids in
  `views/I<N>.md`). Same union pattern as the main board (C02), keyed on the task
  id instead of "parent-less."
- Reuse the `views/<slug>.md` membership file and `TaskBoards.tsx` rendering from
  C01; the only difference is the auto-rule (children of `I<N>`).
- Entry point is a button in the task's detail tab (C01/C04 UI), matching the
  author's "button in the task tab" ask — not a pre-provisioned board.
