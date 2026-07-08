---
id: F02
epic: F — In-app terminals
title: Task-linked session (exports TASKS_TASK_ID)
size: S
requires: [F01]
novel: false
---

## What
Each task's terminal is backed by a tmux session named for that task (e.g.
`task-I<N>`), and that session's environment exports `TASKS_TASK_ID=I<N>`. Any
process the user starts in the terminal — an agent, a script — inherits the task
identity from its environment.

## Why it exists
The terminal is only useful for supervision if the app knows *which task* the
work belongs to. Exporting the id into the session is the small hinge that turns
"a shell" into "a shell that knows what it's working on": it's what later lets
Claude Code hooks report status back onto the right card, auto-move the task to
IN PROGRESS, and record session history per task.

> The task file is the interface to an agent — every session begins *"Please work
> on task /…/I<id>.md"*. Carrying that id in the environment is how the rest of
> the app follows along.

## Acceptance criteria (EARS)
- When the system creates a task's terminal, it shall name the tmux session
  deterministically from the task id (e.g. `task-I<N>`).
- The system shall export `TASKS_TASK_ID` set to the task's id into the session
  environment before any user command runs.
- When the same task's terminal is reattached, the system shall reuse the same
  named session rather than creating a second one.
- A process started inside the terminal shall be able to read `TASKS_TASK_ID`
  from its environment.

## Build notes
- Set the env var when the tmux session is first created (pty manager in main),
  not per-attach, so it survives reattach and reload.
- The session name is the join key for downstream features — keep it a pure
  function of the task id so hooks (H03), auto-IN-PROGRESS (F09), and session
  history (I01) can reconstruct it. When multiple terminals per task arrive
  (F04), the per-task name becomes a prefix and F05 adds the per-terminal id.
- Slug/name matching must be exact: a mismatch between the session name and the
  id downstream features expect silently breaks status routing.
