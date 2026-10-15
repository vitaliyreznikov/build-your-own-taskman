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
| **Overdue** | Due < now — every task, repeating or not |
| **Today** | Due later today |
| **Tomorrow** | Due tomorrow (collapsed) |
| **This week** | Due within the next 7 days (collapsed) |
| **Later** | everything further out (collapsed) |
| **Done today** | occurrences completed today (collapsed) |

Each row shows the due time, task id, title, a 🔁 mark if it repeats (Q03), the
category, and actions: **✓ Done**, **+1h** (snooze one hour, one click), **⏰
Snooze** (15 min / 1 h / 3 h / tomorrow 09:00), and **✕ Skip** on repeating
ones. Snoozing a one-off task moves **both** Start and Due, keeping the gap
between them. Snoozing a repeating task moves **only the current occurrence**:
Start = now + duration and Due = the later of its Due and the new Start. The
schedule is untouched: Done/Skip on the snoozed occurrence advances to the same
next occurrence it would have without the snooze (Q03 keys the occurrence by its
scheduled start, not by the stored Due). A snooze that would reach the next
occurrence's start is refused — skip instead. A row whose Start is
still in the future is dimmed and says "starts <time>". Clicking a row opens the
task's panel. The sidebar entry shows a badge: overdue + due-today count.

## Why it exists
Priority-ranked Kanban boards are the wrong lens for things that must happen at a
time: a P2 pill reminder is more urgent at 09:55 than a P0 project. The Today view
is the time-ordered lens — one place to see everything that is due and tick it off
— while the boards stay the priority lens (time-bound tasks still show there).

**Overdue means Due < now, with no exceptions.** An earlier version kept a
repeating task with no window (`cron:0 7 * * *`) under Today as "due now" until
its next occurrence started. In practice that listed yesterday's 12:00 item
under Today the next morning, next to things due later today, and the user read
the grouping as broken. A windowless repeat is due at its start time; from then
on it is overdue like anything else. Q04 still announces it as "Now" at that
moment, and Q03 still logs it `missed` when the next occurrence starts.

**Repeating tasks can be snoozed.** The first version offered only Skip on a
repeating task, reasoning that its times belong to the schedule. In practice
"take the medicine, but in an hour" is the common case, and Skip loses the
occurrence. Snoozing one occurrence is safe because nothing derives the schedule
from the stored Due (Q03 recovers the occurrence from its scheduled start).

## Acceptance criteria (EARS)
- While any open task's Due is before now, including a repeating task with no
  window, the system shall list it under Overdue and render it as overdue (row,
  card chip, task panel).
- The sidebar shall have a **Today** entry that opens the Today view.
- The Today view shall list every task with a Due that is not DONE or CLOSED, in
  every category, sorted by Due ascending, in the groups above; Tomorrow, This
  week, Later and Done today shall start collapsed.
- While a listed task's Start is in the future, its row shall be dimmed and show
  "starts <time>".
- When the user clicks ✓ on a non-repeating task, the system shall move it to
  DONE (and, if Q03 is built, record a `done` occurrence in its log).
- When the user picks a snooze duration on a non-repeating task, the system shall
  set Start to now + that duration and Due to the new Start plus the previous
  Due − Start gap (zero if Start was empty or after Due).
- Every row shall offer a one-click **+1h** that snoozes the task by 60 minutes.
- When the user snoozes a repeating task, the system shall set Start to now +
  the duration and Due to the later of the current Due and the new Start, and
  shall change nothing else (Repeat, Autorun, the occurrence log).
- When a snoozed repeating occurrence is done or skipped, the system shall
  advance to the next cron occurrence after the occurrence's **scheduled** start,
  so a snooze never shifts later occurrences.
- If the new Start of a repeating snooze would be at or after the next
  occurrence's start, then the system shall refuse the snooze with a message.
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
