---
id: F04
epic: F — In-app terminals
title: Multiple terminals per task
size: M
requires: [F01]
novel: false
---

## What
A single task can have several terminals open at once, each backed by its own
tmux session (e.g. `task-I<N>-1`, `task-I<N>-2`, …). The user can create
additional terminals for a task and kill individual ones, with UI to manage the
set.

## Why it exists
Real work on one task fans out: an agent running in one shell, a dev server in
another, a scratch shell for ad-hoc commands. Because the working style is
"many things in parallel," a task needs to hold more than one live session — and
each must be independently addressable so status and lifecycle track per
terminal, not per task.

> *"Feel free to create subtasks in our system so different processes can work in
> parallel."* The same parallelism applies within a single task's terminals.

## Acceptance criteria (EARS)
- The user shall be able to open more than one terminal for the same task.
- The system shall back each terminal with its own distinct tmux session, named
  with a per-terminal suffix under the task's name (e.g. `task-I<N>-k`).
- The user shall be able to kill an individual terminal (and its tmux session)
  without affecting the task's other terminals.
- When a task with multiple terminals is reopened or the app restarts, the
  system shall reattach to each still-living session (per F01).

## Build notes
- Extend the pty manager to track a set of sessions per task, keyed by the
  per-terminal session name; add a `terminal:kill` IPC to tear down one session.
- The per-terminal session name is what F05 exports as `TASKS_TERM_ID` and what
  the status subsystem keys on — allocate it deterministically and keep it stable
  for the life of the terminal.
- All of F01's reattach hardening applies per session: repaint on re-attach,
  evict stale element on switch, guard the cold-socket race.
