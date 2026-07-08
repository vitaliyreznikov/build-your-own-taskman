---
id: B10
epic: B — Kanban board UI
title: NextAction time model
size: S
requires: [A02]
novel: false
---

## What
A `NextAction` field on each task, a `YYYY-MM-DD HH:MM` timestamp meaning "bring
this task back on my radar at this time." It is used to sort and surface rows so
tasks resurface when they're due.

## Why it exists
Not everything is actionable now; a lot of tasks are "deal with this later."
The NextAction pointer lets a task drop out of attention and come back exactly
when it should, keeping the board focused on what's actually live. It is also
the trigger the P0 native notifications (epic E) are built on.

## Acceptance criteria (EARS)
- When a task has a NextAction time, the system shall use it to order/surface
  that task's row.
- When a task's NextAction time arrives, the system shall surface the task so it
  is back on the user's radar.
- The system shall parse and store NextAction in `YYYY-MM-DD HH:MM` form.

## Build notes
- The field lives in `tasks.md` (A02); parsing/comparison logic lives in
  `time.ts`, consumed by the renderer.
- This is the time model the P0 notifications (epic E) hang off — keep the
  parse/compare logic reusable by electron-main's notifier.
