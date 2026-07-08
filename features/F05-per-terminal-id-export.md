---
id: F05
epic: F — In-app terminals
title: Per-terminal ID export (TASKS_TERM_ID)
size: S
requires: [F04]
novel: false
---

## What
Each terminal's session additionally exports `TASKS_TERM_ID` — the per-terminal
tmux session name (e.g. `task-I<N>-2`). A task can now have several terminals,
and every process knows not just *which task* it belongs to (`TASKS_TASK_ID`)
but *which terminal* it is running in.

## Why it exists
Once a task has multiple terminals (F04), status has to be keyed per terminal or
two agents on the same task would collide onto one icon. `TASKS_TERM_ID` is the
finer-grained join key that lets the app show one status icon per terminal and
one per tab, so the user can supervise several agents on the same task at once.

## Acceptance criteria (EARS)
- When the system creates a terminal, it shall export `TASKS_TERM_ID` set to that
  terminal's unique session name into the session environment.
- The system shall keep `TASKS_TASK_ID` exported alongside it, so both the task
  and terminal identity are available to every process.
- The system shall guarantee `TASKS_TERM_ID` is unique across all live terminals,
  including across different tasks.
- A process started in the terminal shall be able to read `TASKS_TERM_ID` from
  its environment.

## Build notes
- Set both env vars at session-creation time in the pty manager, mirroring F02.
- `TASKS_TERM_ID` is the key the status hooks (H-epic) write against, so one
  Claude Code process reports to exactly one terminal's icon/tab. Keep it equal
  to the tmux session name F04 allocates — don't invent a second identifier.
