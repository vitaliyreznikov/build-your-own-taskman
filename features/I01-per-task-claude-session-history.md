---
id: I01
epic: I — Claude session history & resume
title: Per-task Claude session history
size: M
requires: [F02, H03]
novel: false
---

## What
Every Claude session launched against a task is recorded in a per-task history
file, `.claude-sessions.json`. Each entry captures the session that touched the
task (id, title, timing), building a durable log of which agents worked on it. The
history is shown in the task's detail, hidden behind a disclosure so it doesn't
clutter the note.

## Why it exists
Context should live in files, not in a chat that evaporates. A task is meant to be
a durable, AI-legible workspace an agent can be pointed at cold — and part of that
durability is remembering the agents that already touched it. The session history
turns each task into a record of its own working past, not just its current state.

> The author asked for exactly this continuity: *"We need to save history of all
> claude session that were linked to current task. Show them (hidden by detail)
> and allow to run session again in new terminal."*

## Acceptance criteria (EARS)
- When a Claude session runs against a task, the system shall record it in that
  task's `.claude-sessions.json`.
- Each recorded session shall carry enough identity (session id, title, timing)
  to be listed and later resumed.
- The system shall display a task's session history in its detail view, hidden
  behind a disclosure by default.
- The system shall associate a session with the correct task via the task
  identity exported into the terminal (`TASKS_TASK_ID`).

## Build notes
- The record is written from the hooks (H03) / a sessions helper
  (`sessions-file.mjs`) and read/managed on the app side
  (`claudeSessionsFile.ts`); the terminal's `TASKS_TASK_ID` (F02) ties the
  session to the task.
- Keep the file per-task and human-readable JSON, consistent with the file-as-
  source-of-truth model. It records history, so — unlike the transient status
  file (H02) — it is durable.
- Store enough to drive resume (I02): the session id is what a new terminal will
  reattach to.
