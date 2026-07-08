---
id: E02
epic: E — Native notifications
title: Notification auto-clear
size: S
requires: [E01]
novel: false
---

## What
A pending P0 NextAction notification cancels itself when it is no longer
warranted: when the task moves to DONE or CLOSED, when its priority drops below
P0, or when its NextAction is rescheduled to a different time.

## Why it exists
A notification that fires after the reason for it is gone is noise, and the
author is emphatic about not generating noise. If a task has already been handled
or deprioritized, its alert should quietly disappear rather than nag.

> The author's stance on notification noise: *"no icons blinking. Also send push
> from the app on any attention / done (don't need on start)."*

## Acceptance criteria (EARS)
- When a task with a pending P0 notification is moved to DONE or CLOSED, the
  system shall cancel that notification.
- When such a task's priority drops below P0, the system shall cancel the pending
  notification.
- When such a task's NextAction time is rescheduled, the system shall cancel the
  old scheduled notification (and schedule for the new time per E01).

## Build notes
- Lives with the notifier in Electron main; it must observe the task-state and
  NextAction changes that flow through `tasks:update` and cancel the corresponding
  scheduled timer.
- Track the pending timer per task so a state change can find and clear the right
  one; reschedule = cancel-then-reschedule.
