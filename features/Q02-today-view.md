---
id: Q02
epic: Q — Time-bound tasks
title: Today view (every task with a Due, by time)
size: M
requires: [Q01, B06]
novel: false
---

## What
A sidebar entry **Today** opens a list of every task that has a Due and is not
DONE or CLOSED, ordered by Due and grouped:

| Group | Contains |
|---|---|
| **Overdue** | Due < now (except "due now" repeats, below) |
| **Today** | Due later today |
| **Tomorrow** | Due tomorrow (collapsed) |
| **This week** | Due within the next 7 days (collapsed) |
| **Later** | everything further out (collapsed) |
| **Done today** | occurrences completed today (collapsed) |

Each row shows the due time, task id, title, a 🔁 mark if it repeats (Q03), the
category, and three actions: **✓ Done**, **⏰ Snooze** (15 min / 1 h / 3 h /
tomorrow 09:00) and — on repeating tasks only — **✕ Skip**. A row whose Start is
still in the future is dimmed and says "starts <time>". Clicking a row opens the
task's panel. The sidebar entry shows a badge: overdue + due-today count.

## Why it exists
Priority-ranked Kanban boards are the wrong lens for things that must happen at a
time: a P2 pill reminder is more urgent at 09:55 than a P0 project. The Today view
is the time-ordered lens — one place to see everything that is due and tick it off
— while the boards stay the priority lens (time-bound tasks still show there).

A repeating task with **no window** (`cron:0 7 * * *`) is *due now* from its
occurrence until the next occurrence starts: it is listed under **Today** (not
Overdue) and is not rendered as overdue anywhere. "Show it at 07:00" is the
intent; nothing is late until the next occurrence makes it missed.

## Acceptance criteria (EARS)
- While a repeating task with no window is in its current occurrence, the system
  shall list it under Today and shall not render it as overdue (row, card chip,
  task panel).
- The sidebar shall have a **Today** entry that opens the Today view.
- The Today view shall list every task with a Due that is not DONE or CLOSED, in
  every category, sorted by Due ascending, in the groups above; Tomorrow, This
  week, Later and Done today shall start collapsed.
- While a listed task's Start is in the future, its row shall be dimmed and show
  "starts <time>".
- When the user clicks ✓ on a non-repeating task, the system shall move it to
  DONE (and, if Q03 is built, record a `done` occurrence in its log).
- When the user picks a snooze duration, the system shall set the task's Start to
  now + that duration and shall not change its Due.
- When the user clicks a row outside its actions, the system shall open that
  task's panel.
- The Today entry shall show a badge with the number of listed tasks whose Due is
  in the past or later today; with zero it shall show no badge.
- The view and badge shall update when `tasks.md` changes on disk and when the
  clock crosses a task's Due, without a restart.

## Build notes
- The view does not apply board/category filters: a time-bound task must never be
  hidden by a filter set for planned work.
- "Done today" reads the occurrence log (Q03), so it also covers repeating tasks
  that never leave their column.
- Re-render on a one-minute tick so rows cross from Today to Overdue on time.
