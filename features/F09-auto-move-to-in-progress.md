---
id: F09
epic: F — In-app terminals
title: Auto-move task to IN PROGRESS on terminal open
size: S
requires: [F01, A04]
novel: false
---

## What
Opening a terminal on a task automatically moves that task into the IN PROGRESS
column. The move is idempotent — a task already in IN PROGRESS stays put — and
the system never moves a task *out* of IN PROGRESS on its own.

## Why it exists
Opening an agent terminal *is* the act of starting work, so the board should
reflect that without the user dragging cards around. IN PROGRESS becomes a live,
activity-driven column: it shows what you're actually working on right now, and
the user curates it by removing tasks manually when done.

> *"'IN PROGRESS' is special board. Add task to it every time when user open
> terminal. User will remove tasks manually."*

## Acceptance criteria (EARS)
- When the user opens a terminal on a task, the system shall set that task's
  column to IN PROGRESS.
- If the task is already in IN PROGRESS, the system shall make no change (the
  operation is idempotent).
- The system shall never automatically move a task *out* of IN PROGRESS —
  closing terminals, killing sessions, or agent completion shall not change the
  column.
- The system shall write the column change through the normal task-update path so
  ordering and board membership stay consistent.

## Build notes
- Trigger on terminal *creation*, not on every attach, so reopening a task's
  existing terminal doesn't re-fire.
- Match the task by its exact id/slug from the session; a slug mismatch here
  silently drops the auto-move (a real bug the reference implementation fixed).
- The user owns removal — leaving IN PROGRESS is always a manual action, keeping
  the column an honest "what I chose to still be working on" list.
