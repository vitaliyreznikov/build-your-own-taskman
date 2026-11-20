---
id: Q08
epic: Q — Time-bound tasks
title: Today priority filter, remembered, plus in-row priority change
size: M
requires: [Q02, Q04, Q05, B05, C05]
novel: false
---

## What
The Today view gets a one-click **priority filter** above its list:

| Choice | Shows |
|---|---|
| **P0** | P0 only |
| **≤P1** | P0, P1 |
| **≤P2** | P0, P1, P2 |
| **All** | everything, including P3 and anything with no or an unusual priority |

The filter applies to every row: task occurrences (current and held, Q03) and
subgoal rows (Q05). A subgoal has no priority of its own, so its row is filtered
by **its task's** priority. Repeating tasks are filtered like any other, by
their row's priority.

The choice is **remembered** in a small git-synced file next to `filters.json`,
`today.json`: `{"priority": "P1"}` with `"P0" | "P1" | "P2" | "all"`. It survives
a restart and follows the user across machines; a missing or unreadable file
means **All**, so nothing disappears on a fresh install.

The header says how many rows the filter hides ("3 hidden"); clicking that
switches to **All**.

Every task row (and every subgoal row) shows its **priority chip**. Clicking the
chip opens a small menu P0 / P1 / P2 / P3; picking one writes `Priority` on the
task's row in `tasks.md` (for a subgoal row: on its task, and the menu says so).
If the new priority takes the row out of the filter, the row disappears and a
toast says so, with **Undo** that restores the previous priority.

The filter also governs what Today **counts and announces**:
- the sidebar's overdue badge counts only rows the filter shows;
- Due notifications (Q04 at Start / overdue, Q05 subgoal overdue, and the
  launch summary) fire only for rows the filter shows. A row that went due while
  hidden does **not** fire later when the filter widens or its priority rises —
  it is already late, and the badge and the list show it.

The P0 **Start** notification (E01) is untouched: it is about Start on any P0
task, not about Today.

## Why it exists
Today lists every task with a Due, at any priority, so a long tail of P2/P3
chores buried the few deadlines that matter, and the badge plus notifications
fired for all of them. The user wants to choose, once, how much of that tail
Today shows and pings about — and to fix a wrong priority right where they notice
it, without opening the task.

> *"Постоянный фильтр по приоритету на странице Today … бейдж просроченных и
> уведомления — только по отфильтрованным задачам."*

**One file, not a field in `filters.json`.** `filters.json` maps boards to
category/project/priority selections and is rewritten whole by the board UI.
Today is not a board and its filter is a single threshold, read by the main
process for notifications as well as by the renderer; a separate file keeps the
two writers apart.

**No retro-firing.** Widening the filter could otherwise drop a dozen "Overdue"
banners at once for things the user deliberately hid. Keys of hidden rows are
recorded as already announced, so a notification only ever fires at the moment
the row becomes due (or starts) while visible.

## Acceptance criteria (EARS)
- The Today view shall show a priority filter with the choices P0, ≤P1, ≤P2 and
  All above the list; one click shall select a choice.
- While a choice other than All is selected, the system shall list only task
  occurrences whose task priority is within it, and only subgoal rows whose
  task's priority is within it.
- While All is selected, the system shall list every row, including P3 and rows
  with no recognised priority.
- When the user selects a choice, the system shall write it to `today.json` and
  restore it on the next launch; when `today.json` changes on disk (e.g. git
  sync), the system shall apply it without a restart.
- If `today.json` is missing or invalid, the system shall behave as All.
- While the filter hides at least one open row, the header shall show the hidden
  count, and clicking it shall select All.
- Every task row and subgoal row shall show its (task's) priority; when the user
  picks P0, P1, P2 or P3 from it, the system shall write that `Priority` to the
  task's row in `tasks.md`.
- When an in-row priority change takes a row out of the filter, the system shall
  show a toast with Undo, and Undo shall restore the previous priority.
- The sidebar overdue badge shall count only overdue rows the filter shows.
- The system shall send Due start/overdue notifications (Q04), subgoal overdue
  notifications (Q05) and the launch summary only for rows the filter shows.
- When the filter narrows or a task's priority drops out of it, the system shall
  close that row's open Due notification.
- When a row becomes visible after it already started or went overdue, the
  system shall not notify for that moment.
- The P0 Start notification (E01) shall not depend on the Today filter.

## Build notes
- The predicate is one pure function shared by renderer and main:
  `passesTodayFilter(priority, choice)`, where `rank(P0)=0 … P3=3` and an
  unknown priority passes only All.
- Main reads `today.json` on each scheduler run (cheap) and watches it to
  re-run; the renderer gets it over IPC with a change event, like `filters.json`.
- In the scheduler, mark hidden rows' keys as fired without notifying and leave
  them out of the "live" set — that gives both "close on narrow" and "no
  retro-fire" for free.
