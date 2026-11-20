---
id: P10
epic: P — Agent-facing control API
title: "`create-task --run` + worker brief — file a task and start its worker in one approved call"
size: M
requires: [P02, P03, P04, L03, J02, O02]
novel: true
---

## What
Two additions to the control API that collapse the most common agent flow —
*capture a piece of work, then hand it to its own worker with a specific brief* —
into **one call and one Approve**.

### 1. `create-and-run` — create a task and run it

```
taskman create-task "<title>" --run [--body <text>] [--category <name>]
    [--priority P0|P1|P2] [--due "YYYY-MM-DD HH:mm"] [--type v1|v2]
    [--prompt <brief>] [--agent claude|codex|devin]
```

- `action = "create-and-run"`, `params = { title, body?, category?, priority?,
  due?, type?, prompt?, agent? }`. A separate action, not new parameters on
  `create-task`: P03's wire contract (title + body only) stays exactly as it was,
  and `--run` is purely the CLI spelling.
- The row fields allowed here are the ones that must be right **before the worker
  starts**, so they cannot be left to "edit the files afterwards":
  - `category` — decides which Claude account the worker launches on (J02) and is
    part of what the worker reads first;
  - `priority`, `due` — what the worker should treat as urgency;
  - `type` — v1 note or v2 document (O02); the worker's launch prompt is built
    from it, so switching formats after launch would brief the worker for the
    wrong file.
- Deliberately **not** parameters: `column` (running a task moves it to IN
  PROGRESS anyway, F09), `start` (a future Start contradicts "run now"), parent,
  project, relations, autorun, repeat — those still go through file edits, as in
  P03.
- Without `--run`, the CLI keeps P03's surface and refuses the row flags with a
  pointer to either `--run` or editing `tasks.md`.

Approval sentence names everything at once, so one Approve is an informed one:
*"Session in `<caller>` wants to create and run a task: "`<title>`" [`bolt` · `P1`
· due `2026-11-21 10:00` · v2] — brief: "`<first ~160 chars of prompt>`…""*.

On approve, the executor:
1. creates the task through P03's create path with the given row fields
   (defaults for the rest), writes `body` into the note (v1) or into the
   document's `body_human` (v2);
2. opens the new task's terminal through P02's open path with the agent
   auto-started, the work-on-task prompt typed, and `prompt` appended to that
   same first submission;
3. resolves `done` with `result = { taskId, notePath, sessionName }`
   (`notePath` is `I<N>.md` or `I<N>.json` per type).

If step 2 fails, the task created in step 1 is kept and the request resolves
`error` with a message naming the created `I<N>`, so the caller can retry with
`taskman run I<N>`.

The over-budget agent chooser (F19) stands in for the Approve banner exactly as
it does for `open-terminal`.

### 2. `--prompt` on `run` — brief a worker on an existing task

```
taskman run <taskId> [--goal <sg>] [--new] [--agent …] [--prompt <brief>]
```

- `open-terminal` gains an optional `prompt` param.
- When it opens a terminal (none live for the target, or `--new`), the brief is
  appended to the work-on-task / work-on-subgoal prompt — one agent turn, not two.
- When a live terminal for that exact target already exists and `--new` is not
  given, P02 would only focus it; with a brief, the executor instead **delivers
  the brief into that live session** through L03's courier (headed `[TaskMan]
  brief from <caller>`), and resolves with `{ sessionName, delivered: true }`.
  A failed delivery resolves `error` (the brief was not delivered) rather than
  silently dropping it.
- The approval sentence appends *"— brief: "…""*.

## Why it exists
Filing a task and starting its worker took four steps and two approvals: P03
`create-task` → Approve → hand-edit the `tasks.md` row (category, priority, due) →
P02 `run` → Approve → wait for the worker to register as a peer → message it the
actual brief, because the worker only gets the generic work-on-task prompt. An
intake session (F26) does exactly this many times a day. Each step is a chance to
stall (the second approval sits unseen, the brief races the worker's first turn),
and the category edit has to land *before* the run or the worker starts on the
wrong account.

One call with one approval removes the stalls and makes the order of operations
the app's problem, not the caller's. The brief riding inside the first submission
removes the race. P03's "title only, edit the rest" stays the right answer for
*filing* a task; this verb exists for *filing and launching*, where the few fields
that steer the launch must be known before it.

## Acceptance criteria (EARS)
- The system shall register a `create-and-run` action taking a required `title` and
  optional `body`, `category`, `priority`, `due`, `type`, `prompt` and `agent`.
- The system shall reject a `create-and-run` request whose `title` is blank, whose
  `category` is not in the category list, whose `priority` is not a known
  priority, whose `due` is not a `YYYY-MM-DD HH:mm` stamp, whose `type` is not
  `v1`/`v2`, or whose `agent` is not a known agent — before any approval prompt.
- The approval description for `create-and-run` shall name the caller, the title,
  every given row field, and the start of the brief.
- When `create-and-run` is approved, the system shall create the task via the same
  create path as P03 with the given fields applied, and default the rest.
- When a `body` is given, the system shall write it into the note of a v1 task and
  into `body_human` of a v2 task's document.
- When the task has been created, the system shall open its terminal via the same
  open path as P02, auto-start the agent, and send the work-on-task prompt with
  `prompt` appended in the same first submission.
- The opened terminal shall run on the account the task's category maps to (J02),
  unless the agent chooser (F19) picked one.
- When the request resolves `done`, its result shall carry the new task id, its
  note/document path and the opened session name.
- If opening the terminal fails after the task was created, the system shall keep
  the task and resolve `error` with a message naming its id.
- The `create-task` action shall keep accepting only `title` and `body`.
- When `open-terminal` with a `prompt` opens a terminal, the system shall append the
  prompt to that terminal's launch prompt.
- When `open-terminal` with a `prompt` targets a task or subgoal whose terminal is
  already live and `new` is not set, the system shall deliver the prompt into that
  live session instead of opening a second terminal, and resolve with
  `delivered: true`.
- If that delivery fails, the system shall resolve the request `error`.
- The CLI `taskman create-task` shall send `create-and-run` when `--run` is given,
  and shall refuse the row flags without `--run`.

## Build notes
- Server: one `registerApiCommand('create-and-run', …)` next to `create-task`. Its
  `execute` calls the existing `createTaskInternal({title, category, priority,
  due, type})`, seeds the body, pushes `tasks:changed`, then
  `ctx.exec({...req, action: 'open-terminal', params: {taskId, prompt, agent}})`
  — the renderer's existing open-terminal branch, with the same `requestId` so a
  chooser pick (F19) keyed on it still applies.
- The launch prompt already has an `extraPrompt` slot (L01/O07 use it for PR
  reactions); `prompt` maps onto it — no new prompt plumbing.
- Live-target check for `run --prompt`: task-level = the task's *own* terminals
  only (not its subgoal terminals), matching what P02 would focus; subgoal-level =
  that subgoal's terminals. Delivery = L03's `deliverToLiveSession`.
- Cap the brief at a sane size (e.g. 8 KB) in `validate`; it is typed into a TUI.
- Over-budget chooser: include `create-and-run` in the F19 condition; the label is
  the title (the id doesn't exist yet).
