---
id: F12
epic: F — In-app terminals
title: Auto-start Claude with the work-on-task prompt
size: M
requires: [F02]
novel: false
---

## What
When a new task terminal is created, the app can automatically launch the coding
agent and seed it with a preconfigured prompt — so opening a terminal on a task
drops you straight into an agent already told what to do. Concretely: it types
`claude`, waits for the CLI to become ready, then types a **work-on-task prompt**
that names the task's Markdown file and restates the status-marker contract:

> `Please work on task I<N>.md. Report status inside your answers: … "✅ done:" /
> "⏳ waiting:" / "⚠️ attention:" …`

When the terminal is instead **resuming** a prior session, it runs
`claude --resume <sessionId>` and does *not* re-type the prompt — the restored
conversation already carries its instructions. The whole behavior is gated by a
single config flag so it can be turned off.

### The exact prompt (from the reference implementation)
The app types `claude`, waits until it looks ready, then types this verbatim
(`<N>` is the task id):

```
Please work on task I<N>.md. Report status inside your answers: when your phase changes add a line "🤖 working: <what you're doing>", and end every turn's final message with one marker line — "✅ done: <one-line summary>", "⏳ waiting: <external process: job, CI, review — never waiting on me>" or "⚠️ attention: <anything you need from me>" (a question = attention, not done).
```

This single seed does double duty: it hands the agent its task (the `.md` file)
*and* installs the status-marker protocol the supervision layer (H03/H04) reads
back out of the transcript.

## Why it exists
The core loop is "point an agent at a task." Making the human type `claude` and
paste the same boilerplate every time is friction repeated dozens of times a day.
Auto-starting with the task file *and* the status contract also guarantees the
agent will emit the markers the status system (H03/H04) depends on — the
convenience feature and the supervision feature reinforce each other.

> The author's sessions all begin exactly this way — "Please work on task
> /…/I<id>.md" — and the app encodes that ritual so every new terminal starts
> mid-workflow instead of at a bare shell.

## Acceptance criteria (EARS)
- When a new (non-resumed) task terminal is created and auto-start is enabled, the
  system shall launch the agent CLI and then type the work-on-task prompt for that
  task.
- The prompt shall reference the task's own note file (`I<N>.md`) and restate the
  status-marker contract.
- When the terminal is resuming a saved session, the system shall issue a resume
  command and shall not re-type the work-on-task prompt.
- When the agent CLI does not report ready within a bounded timeout, the system
  shall type the prompt anyway (terminal input queues) rather than hang.
- When auto-start is disabled by config, the system shall open a plain shell and
  type nothing.

## Build notes
- Readiness is heuristic: after typing `claude`, poll the pane (e.g. tmux
  `capture-pane`) until it "looks ready," with a hard deadline (~15s). On timeout,
  type the prompt regardless — the tty line discipline queues input and it's
  visible on screen either way. Abort quietly if the session dies mid-wait.
- Give the shell a short beat (~400ms) to finish initializing before typing.
- Keep the seeded status-marker text in sync with the agent guide (AGENTS.md);
  repeating it in the prompt keeps marker compliance high, which is what lets the
  hooks (H03) drive the card icon.
- The resume path depends on session history (I01/I02) to supply the session id;
  base auto-start needs only the task-linked terminal (F02).
- Gate behind a boolean config (the reference app: `AUTO_CLAUDE_TASK_TERMINALS`)
  so users who prefer a manual shell can opt out.
