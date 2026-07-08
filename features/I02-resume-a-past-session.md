---
id: I02
epic: I — Claude session history & resume
title: Resume a past session in a new terminal
size: M
requires: [I01]
novel: false
---

## What
From a task's session history, the user can pick a prior Claude session and
resume it in a **new terminal** — reopening that conversation with its context
intact, rather than starting cold. One click on a history entry spawns a terminal
that continues where that agent left off.

## Why it exists
The value of remembering sessions (I01) is only realized when you can pick one
back up. Work on a task rarely finishes in a single sitting; being able to resume
the exact agent that was mid-investigation — instead of re-explaining everything —
is what makes the task a continuous workspace across days and across many agents.

> The author's request tied history and resume together in one breath: *"Show them
> (hidden by detail) and allow to run session again in new terminal."*

## Acceptance criteria (EARS)
- When the user selects a session from a task's history, the system shall open a
  new terminal that resumes that Claude session.
- The resumed session shall continue the prior conversation's context rather than
  starting fresh.
- The system shall run the resume in a new terminal for the task, leaving other
  terminals untouched.
- When a recorded session can no longer be resumed, the system shall surface that
  clearly rather than opening an empty or broken terminal.

## Build notes
- Resume spawns a new task terminal (F-epic) and invokes Claude with the stored
  session id from `.claude-sessions.json` (I01); the terminal exports the usual
  `TASKS_TASK_ID` / `TASKS_TERM_ID` so status and history keep working for the
  resumed run.
- Treat resume as "new terminal, continued conversation" — do not try to reattach
  into an existing terminal; that keeps the terminal lifecycle simple and
  consistent with multi-terminal semantics.
- A resumed session is itself a session on the task, so it should be recorded
  back into the history (I01) like any other.
