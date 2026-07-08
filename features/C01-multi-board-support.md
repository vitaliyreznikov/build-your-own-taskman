---
id: C01
epic: C — Boards / views / filters
title: Multi-board support (views.md + views/)
size: M
requires: [A02, B01]
novel: false
---

## What
Beyond the single main Kanban board, the app supports any number of additional
board surfaces. The set of boards lives in `views.md`, and each user-created
board has its own membership file at `views/<slug>.md`. A board is a *view* onto
the same global set of tasks — it selects which tasks it shows, but it does not
own their state.

## Why it exists
One flat board stops scaling once you run many parallel tasks with subtasks. The
author wanted to slice the same task set several ways — a hand-curated board, or
an automatic board for a task and its children — without duplicating tasks or
forking their state.

> In the author's words: *"Now we have one, main kanban board. I want to have
> additional boards of two types: boards I create / boards for any task with
> subtasks. ... All tasks go to main board by default but children tasks."*

## Acceptance criteria (EARS)
- The system shall read the list of boards from `views.md` and render one board
  surface per entry.
- When the user creates a board, the system shall write a new `views/<slug>.md`
  membership file for it.
- Each board shall determine its visible tasks from its membership file plus its
  automatic rules (see C02, C03), reading task state from the global index (A02).
- The system shall not require boards to be created in advance; a board comes
  into existence only when created on demand.
- When a task's column/priority/other state changes, the change shall be visible
  identically on every board that shows the task (state is global, not per-board).

## Build notes
- Two on-disk pieces: `views.md` (the registry of boards, parsed by
  `viewsFile.ts`) and one `views/<slug>.md` per user board holding its member
  list. Rendering is `TaskBoards.tsx`.
- Membership is *additive*: a task can appear on any number of boards at once.
  The board file lists member ids; it never stores task fields (column, priority)
  — those always come from `tasks.md` (A02).
- Reserve `views/main.md` for the main board's pinned children (C02) and
  `views/I<N>.md` for per-task boards (C03) so all board-membership files share
  one directory and one parser.
- Slugs should be stable and filesystem-safe; keep the slug → file mapping
  trivial the same way A01 keeps id → filename trivial.
