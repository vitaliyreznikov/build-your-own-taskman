---
id: Q04
epic: Q — Time-bound tasks
title: Due notifications (at Start and on overdue, any priority)
size: S
requires: [Q01, Q02, E01, E03]
novel: false
---

## What
A time-bound task (one with a Due, not DONE/CLOSED) gets native notifications
regardless of priority:

1. **At Start** — "<title> — due <time>", when the task becomes workable (only if
   it has a Start).
2. **At Due** — "Overdue: <title>", if it is still open.

Each notification offers **Done** and **Snooze 1h** (repeating tasks: **Done**
and **Skip**) buttons where the OS supports
notification actions; clicking the body opens the Today view with the task
selected. On launch, instead of one banner per already-overdue task, the system
shows a single summary ("3 overdue, 2 due today").

## Why it exists
E01 notifies only P0 tasks (and, since Q04, only those without a Due), because planned work at lower priority does not need
an interrupt. Time-bound tasks are the opposite: a P2 "take medicine" must
interrupt at its time. Acting from the banner closes the loop without switching to
the app.

## Acceptance criteria (EARS)
- When a time-bound task's Start arrives, the system shall show a notification
  naming the task and its Due, at any priority.
- When a time-bound task's Due passes while it is open, the system shall show an
  overdue notification for it, once per occurrence; for a repeating task with no
  window the notification shall read "Now: <title>" instead of "Overdue".
- The E01 P0 notification shall not fire for a task that has a Due — Q04 owns
  its notifications, so a P0 chore gets one banner (with Done / Snooze), not two.
- When the user clicks Done on a notification, the system shall mark the task done
  exactly as the Today view's ✓ does (Q02/Q03).
- When the user clicks Snooze 1h, the system shall snooze the task exactly as the
  Today view does (Q02: Start and Due both move); a snoozed task notifies again at
  its new times.
- When the user clicks Skip on a repeating task's notification, the system shall
  skip the occurrence (Q03).
- When the user clicks the notification body, the system shall show the window,
  open the Today view and select the task.
- When the app starts with time-bound tasks already overdue or started, the system
  shall show one summary notification instead of one per task.
- When a task is done, skipped, advanced, rescheduled or deleted, the system shall
  close its pending notification and re-arm for the new times.

## Build notes
- Reuse E03's chunked timers for far-future times.
- Notification action buttons need the app's notification style set to Alerts on
  macOS to stay visible; without actions support the body click is the fallback.
