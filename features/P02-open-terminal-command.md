---
id: P02
epic: P — Agent-facing control API
title: "`open-terminal` command — an agent asks TaskMan to open a task (or subgoal) terminal"
size: M
requires: [P01, F12, F09, O09]
novel: false
---

## What
The first real verb on the control API (P01): an agent asks the app to **open a
terminal for a task**, optionally scoped to a **subgoal**, and — once the user
approves — the app opens it and points a fresh agent at that work.

CLI shape:

```
taskman open-terminal <taskId> [--goal <subgoalId>] [--agent claude|devin]
```

- `action = "open-terminal"`, `params = { taskId, subgoalId?, agent? }`.
- The terminal **always auto-starts the agent** (there is no bare-shell mode on
  this verb): with no `--goal` it opens the task terminal and types the
  work-on-task prompt (F12); with `--goal` it opens the **per-subgoal terminal**
  (O09) and types the work-on-subgoal prompt.
- `--agent` is optional; omitted, the app uses its normal agent selection (the
  over-budget Claude/Devin logic, F15) exactly as a manual open would.

Approval sentence (P01's `describe`): *"Session in `<source.taskId>` wants to open a
terminal for `<taskId>`"* — with *" › `<subgoalId>`"* appended when a goal is
given, and *"(no caller)"* standing in when `source.taskId` is absent.

On approve, the executor performs the **same open path a manual click uses**:
`openSession({ name, taskId, subgoalId, subgoalText, autoClaude: true, agent })`
followed by the move to IN PROGRESS (F09). Because opening a terminal is a
renderer/xterm operation, the executor delegates to the renderer the same way
autorun's fire does (main sends a message; the renderer calls the existing
`openTaskTerminal` / `openSubgoalTerminal` store action) rather than calling
`openSession` directly. The request resolves `done` with
`result = { termId, sessionName }`.

If a live terminal for that exact target already exists, the executor does not open
a second one — it resolves `done` with the existing terminal's ids and the approval
UI/notification says so, matching autorun's "already open → just surface it"
behaviour.

## Why it exists
This is the concrete thing that motivated the whole API: an agent working task
`I244` realises the next move belongs to `I353` (or to a specific subgoal of it)
and wants to spin up a dedicated agent there — but it cannot reach into the app to
open a terminal, and it must not do so without the user's say-so. `open-terminal`
gives it exactly that: a request the user sees and approves, after which the app
opens a first-class task/subgoal terminal (stable tab id, `TASKS_TASK_ID`, board
membership, per-terminal status keying) with an agent already running the right
prompt. It reuses every piece of the manual and autorun open paths, so the opened
terminal is indistinguishable from one opened by hand — it just started as an
agent's approved request.

## Acceptance criteria (EARS)
- The system shall register an `open-terminal` action taking `taskId` and optional
  `subgoalId` and `agent`, and shall reject a request whose `taskId` names no task.
- When `open-terminal` is approved without a subgoal, the system shall open the
  task's terminal, auto-start the agent, and type the work-on-task prompt (F12),
  and move the task to IN PROGRESS (F09).
- When `open-terminal` is approved with a subgoal, the system shall open that
  subgoal's per-subgoal terminal (O09), auto-start the agent, and type the
  work-on-subgoal prompt.
- When an `agent` is specified, the system shall launch that agent; when it is
  omitted, the system shall use the app's normal agent selection (F15) unchanged.
- The approval description shall name the requesting session, the target task, and
  the subgoal when present.
- When the request resolves `done`, its result shall carry the opened terminal's id
  and session name.
- When a live terminal already exists for the requested target, the system shall not
  open a second terminal and shall resolve with the existing terminal's ids.
- The opened terminal shall be identical in identity and behaviour (env exports,
  board membership, status keying, agent prompt) to one opened manually for the same
  target.

## Build notes
- This verb is the canonical example of P01's "executor delegates to the renderer"
  note: main resolves the request only after the renderer confirms the open, so the
  `result` can carry the real `termId`.
- Reuse the existing `TerminalOpenOptions` fields (`taskId`, `subgoalId`,
  `subgoalText`, `autoClaude`, `agent`) verbatim — no new open-path plumbing. The
  subgoal text needed for the prompt is looked up from the task document the same
  way the manual subgoal-open does (O09), so the caller only passes the subgoal id.
- Duplicate-terminal detection reuses the same "is there a live session for this
  target" check autorun already performs before firing.
- Validation only needs to confirm the task (and, if given, the subgoal) exists;
  everything else is the standard open path's responsibility.
