---
id: L03
epic: L — PR-review blocking
title: Deliver a trigger into the task's live agent session (a helper agent sends it)
size: M
requires: [L01, J03, H08]
novel: true
---

## What
When an external trigger fires for a task that **already has a live terminal**
— a PR became actionable (L01/O14), a monitor fired (L02) — the app no longer
just shows a notification. It **delivers the trigger's prompt into the agent
session running in that terminal**, as a message the agent reads and acts on at
its next turn boundary.

Delivery uses the agent's own peer-messaging, not keystrokes: the app starts a
short-lived **headless helper agent** (`claude -p`, the cheapest model) on the
**same Claude account** the target terminal runs on, and tells it to find the
peer session by its terminal (tmux) name and send it the prompt verbatim with
its session-messaging tool. The helper prints `SENT <peer>` or `FAIL <reason>`
and exits.

- **Which terminal:** for a task-level trigger, the task's live terminals; for a
  subgoal trigger (O07), that subgoal's own terminal (O09). With several, the one
  whose agent status (H08) changed most recently.
- **Which account:** the account the running agent actually uses (J03's
  `configDir` from the status file), else the terminal's remembered account,
  else the default. Peer discovery is per account — a helper on the wrong account
  sees no peers.
- **Fallback:** if there is no agent running in the terminal (just a shell), the
  helper reports `FAIL`, times out, or exits non-zero, the app shows the
  notification it used to show. A successful delivery also shows a quiet
  notification ("delivered to I665's session") so the human knows the agent was
  woken.

The delivered message is the same prompt the trigger would have appended when
opening a fresh terminal, prefixed with a one-line header naming the trigger
(`[TaskMan] PR approved: …` / `[TaskMan] monitor fired: …`), so the agent can
tell it apart from the human typing.

## Why it exists
"Already has a terminal" is the common case, not the exception: a task waiting
on a PR or a monitor usually still has the session that opened the PR or armed
the monitor. A notification there means the trigger dies unless the human is
watching — the opposite of the point of L01/L02 when the event lands at night.
Typing into the pane would collide with whatever the human is typing and with a
running turn; the agent's own peer-messaging queues the message and hands it
over at a turn boundary, which is exactly the semantics of "another teammate
tells you something".

## Acceptance criteria (EARS)
- When an L01/O14 PR reaction or an L02 monitor fire is due for a task (or
  subgoal) that has a live terminal, the system shall deliver the trigger's
  prompt into the agent session running in that terminal instead of only
  notifying.
- The system shall deliver by running a headless helper agent that addresses the
  target session by its terminal name and sends the prompt through the agent's
  peer-messaging; the system shall not type the prompt into the terminal.
- The system shall run the helper on the Claude account the target terminal's
  agent uses.
- The system shall run the helper outside any task context, so it never shows as
  a task's agent status or session history.
- When delivery fails (no agent in the terminal, helper reports failure, times
  out or errors), the system shall show the notification the trigger showed
  before this feature.
- When delivery succeeds, the system shall show a notification saying which
  task's session received it.
- The delivered message shall start with a header line naming the trigger.

## Build notes
- **Helper invocation.** `claude -p --model haiku --allowedTools ListAgents SendMessage`
  with the prompt on **stdin** (`--allowedTools` is variadic and swallows a
  trailing positional prompt). Env: the target's `CLAUDE_CONFIG_DIR` (unset for
  the default account); **strip** `TASKS_TASK_ID`, `TASKS_TERM_ID` and the
  `CLAUDECODE` / `CLAUDE_CODE_*` vars inherited from the app, so the status hooks
  don't attribute the helper to a task and it isn't treated as nested. `cwd` a
  temp dir. Timeout ~150 s.
- **Helper prompt.** "Call ListAgents. Find the session whose tmux name is exactly
  `<session>`. SendMessage it exactly the text between the markers … Print
  `SENT <name>` or `FAIL <reason>`." Parse the last `SENT`/`FAIL` line; anything
  else is a failure.
- **Serialize** helpers (one at a time) — they are cheap but not free, and a
  burst of triggers should not spawn a burst of agents.
- **Live-terminal lookup** reuses the existing test (tmux session list + P05
  rebinds + O09 subgoal slug); it now returns the session name(s), not a bool.
- This is the reference implementation's choice of transport. Any agent with a
  peer-messaging / inbox mechanism works the same way; without one, fall back to
  the notification.
