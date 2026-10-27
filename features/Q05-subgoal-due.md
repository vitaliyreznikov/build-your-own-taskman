---
id: Q05
epic: Q — Time-bound tasks
title: Subgoal Due — v2 subgoals with a deadline show in Today
size: M
requires: [Q02, Q04, O03, O15]
novel: false
---

## What
A v2 subgoal may carry a **`due`** — `"YYYY-MM-DD HH:mm"` local, one-off (no
repeat) — meaning *this part must be done by then*. Every unfinished subgoal with
a `due`, under a task that is not DONE/CLOSED, is listed in the **Today** view
next to the tasks, in the same time groups, as `I522 · g4 — <subgoal title>` with
the task title muted. From Today: **✓** marks the subgoal `done`; **+1h** / **⏰**
move its `due` that far from now. Overdue subgoals count in the Today badge and
get the Q04 overdue notification (Done / Snooze 1h). In the subgoal tree the
`due` shows as a ⏱ chip (red when overdue) next to the O15 date, and the O15
"by date" sort uses the earlier of `due` and `nextAction`.

## Why it exists
A big task is a list of parts with their own deadlines (a declaration by
31.01, a payment by the 15th). With only task-level Due, those deadlines lived in
subgoal titles, invisible to Today.

> *"нужно ввести понятие due date для подзадачи. В итоге подзадачи с due date
> должны появляться в Today так же как задачи"*

## Acceptance criteria (EARS)
- The system shall read an optional `due` on every subgoal and keep it through
  normalization; an absent or empty `due` means none.
- While a subgoal is not `done`, has a `due`, and its task is not DONE or CLOSED,
  the system shall list it in Today in the group its `due` falls in, sorted by
  time together with the tasks.
- When the user presses ✓ on a subgoal row, the system shall set that subgoal's
  `state` to `done` in the document and drop the row.
- When the user snoozes a subgoal row by N minutes, the system shall set its
  `due` to now + N minutes.
- When the user clicks a subgoal row, the system shall open its task.
- The Today badge shall count overdue subgoals as well as overdue tasks.
- When a subgoal's `due` passes while the app runs, the system shall show one
  overdue notification for it, with Done / Snooze 1h actions; it closes when the
  subgoal is done or its `due` moves.
- The system shall show a ⏱ `due` chip on an unfinished subgoal's row, styled
  overdue once it has passed.
- Writes shall touch only the one subgoal's `due` / `state` keys, read-modify-
  write on the current file (the K02 rule).

## Build notes
- Collect `{subgoalId, title, due}` for unfinished subgoals at document-index
  time, like K02 autoruns, and ship them in the fresh state; Today filters by
  the task's Column.
- Subgoal `due` has no Start and no Repeat; a recurring deadline stays a task.
