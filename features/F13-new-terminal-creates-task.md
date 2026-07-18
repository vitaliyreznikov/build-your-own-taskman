---
id: F13
epic: F — In-app terminals
title: "+" new-terminal button spawns a task-creating agent
size: M
requires: [F06, F12, A07]
novel: true
---

## What
The "+" button in the terminal tab bar no longer opens a bare shell. Instead it
opens a fresh terminal, auto-starts the coding agent (the F12 machinery), and
seeds it with a **create-a-new-task prompt** rather than a work-on-task prompt.
The agent interviews the user for a few details, distills a clear title and short
description, then *creates the task* — allocating the next free `I<N>` id (A07),
adding its row to the central index (`tasks.md`), and writing the `I<N>.md` note
— following the repo's agent guide (`AGENTS.md`). It then immediately begins
working on the task it just created, in the same session.

Because there is no task yet, this path is the one auto-start case that is *not*
gated on a `taskId`: the terminal is a plain `term-<n>` session with no
`TASKS_TASK_ID`. The seed prompt also restates the status-marker contract so the
supervision layer (H03/H04) can drive the tab/card icon once the agent starts the
work.

### The exact prompt (from the reference implementation)
After typing `claude` and waiting until it looks ready, the app types this
verbatim:

```
The user wants to create a new task in TaskMan (this repo) and start solving it at the same time. Ask them for a few details — enough to identify a clear title and a short description — then create the task (allocate the next free I<N> id, add its row to tasks.md, write the I<N>.md note) following the conventions in AGENTS.md, and immediately begin working on it in this same session. Report status inside your answers: when your phase changes add a line "🤖 working: <what you're doing>", and end every turn's final message with one marker line — "✅ done: <one-line summary>", "⏳ waiting: <external process: job, CI, review — never waiting on me>" or "⚠️ attention: <anything you need from me>" (a question = attention, not done).
```

## Why it exists
Capturing a new task and starting on it were two separate motions: hit a quick-add
bar (B04) to file the task, then open its terminal (F12) to point an agent at it.
For the common "I just thought of something and want to act on it now" case, the
"+" button — already the most reachable control on the terminal view — collapses
both into one gesture. The agent both *files* the work (so it exists on the board
with a real id and note) and *starts* it, so the human goes from an idea to an
agent mid-task in a single click, without hand-typing `claude` or a prompt.

## Acceptance criteria (EARS)
- When the user clicks the new-terminal ("+") button, the system shall open a new
  non-task terminal, launch the agent CLI, and type the create-a-new-task prompt.
- The create-a-new-task prompt shall instruct the agent to elicit a title and
  description, allocate the next free task id, write the index row and the note
  file per the agent guide, and then start working on the created task.
- The create-a-new-task prompt shall restate the status-marker contract so the
  agent emits the markers the supervision layer reads.
- While auto-start's readiness/timeout handling (F12) shall apply unchanged, the
  create-task path shall fire even though the terminal has no linked task id.
- When the agent CLI does not report ready within the bounded timeout, the system
  shall type the create-a-new-task prompt anyway rather than hang.

## Build notes
- Reuse the F12 auto-start plumbing: the only additions are (a) a `createTask`
  option on the terminal-open request, (b) relaxing the auto-start gate so it
  fires on `autoClaude || resume` when `taskId || createTask` (previously
  `taskId` was mandatory), and (c) selecting the create-task prompt instead of the
  work-on-task prompt when `createTask` is set and there is no task id.
- The button's own click handler is the only caller that sets `createTask`; every
  other auto-start path still passes a `taskId` and gets the work-on-task prompt.
- Id allocation, the index row, and the note file are produced *by the agent*
  following `AGENTS.md`, not by app code — keep the mechanics in the guide (A07)
  so the one source of truth stays the guide the agent reads.
- Since the terminal starts with no `taskId`, it does not join the IN PROGRESS
  board on open (F09); board membership follows once the agent has created the
  task and, if configured, opens/keeps a task-linked terminal for it.
- Keep the seeded status-marker text in sync with the agent guide, same as F12.
