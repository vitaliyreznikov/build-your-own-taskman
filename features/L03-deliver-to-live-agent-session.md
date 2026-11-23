---
id: L03
epic: L — PR-review blocking
title: Deliver a trigger into the task's live agent session (a helper agent sends it)
size: M
requires: [L01, J03, H08, K01, F23]
novel: true
---

## What
When an external trigger fires for a task that **already has a live terminal**
— a PR became actionable (L01/O14), a monitor fired (L02) — or a **scheduled
autorun** comes due (K01 task, K02 subgoal, Q03 `each`) — the app no longer
just shows a notification. It **delivers the trigger's prompt into the agent
session running in that terminal**, as a message the agent reads and acts on at
its next turn boundary.

Delivery uses the agent's own peer-messaging, not keystrokes: the app starts a
short-lived **headless helper agent** (`claude -p`, the cheapest model) on the
**same Claude account** the target terminal runs on, and tells it to find the
peer session by its terminal (tmux) name and send it the prompt verbatim with
its session-messaging tool. The helper prints `SENT <peer>` or `FAIL <reason>`
and exits.

- **Which terminal:** for a subgoal trigger (O07), that subgoal's own terminal
  (O09). For a task-level trigger (a task-level PR blocker, a monitor armed on
  the task, a task autorun), the **task's own terminal** (`task-<id>`, or a
  session re-bound to the task, P05) — never a subgoal terminal while the task's
  own terminal is live, however busy the subgoal worker is. Only when no task
  terminal is live does it fall back to the task's most recently active subgoal
  terminal, and the delivered message then says that it was routed there because
  the task terminal isn't open. With several candidates of the same kind, the
  one whose agent status (H08) changed most recently.
- **Which account:** the account the running agent actually uses (J03's
  `configDir` from the status file), else the terminal's remembered account,
  else the default. Peer discovery is per account — a helper on the wrong account
  sees no peers.
- **No agent there → start one, don't drop the trigger.** If the terminal is
  asleep (F23) it is woken first. If it is only a shell with no hibernation
  record (the agent exited, or the record was lost in a restart/restore), the
  app wakes it anyway: it resumes the terminal's last known conversation
  (`claude --resume <id>`, id from the status file or the F14 snapshot of that
  terminal) and then delivers; with no usable id it starts a fresh agent whose
  first prompt is the usual work-on-task / work-on-subgoal prompt followed by the
  trigger. It never types into a pane whose foreground program is not a plain
  shell (an editor, a running command).
- **Fallback:** only when the trigger still can't be handed to an agent — the
  pane is busy with another program, the agent never comes up, the helper
  reports `FAIL`, times out, or exits non-zero — the app shows the notification
  it used to show, titled so it's clear the trigger was **not** delivered, with
  the session and the reason. A successful delivery also shows a quiet
  notification ("delivered to I665's session") so the human knows the agent was
  woken.

The delivered message is the same prompt the trigger would have appended when
opening a fresh terminal, prefixed with a one-line header naming the trigger
(`[TaskMan] PR approved: …` / `[TaskMan] monitor fired: …` /
`[TaskMan] scheduled autorun (cron): …`), so the agent can tell it apart from the
human typing. For a scheduled autorun the prompt is the work-on-task (or
work-on-subgoal, O09) prompt the fire would have typed into a fresh terminal.

## Why it exists
"Already has a terminal" is the common case, not the exception: a task waiting
on a PR or a monitor usually still has the session that opened the PR or armed
the monitor, and a task armed to run every morning usually still has
yesterday's tab open. A notification there means the trigger dies unless the
human is watching — the opposite of the point of L01/L02 when the event lands at night.
Typing into the pane would collide with whatever the human is typing and with a
running turn; the agent's own peer-messaging queues the message and hands it
over at a turn boundary, which is exactly the semantics of "another teammate
tells you something".

## Acceptance criteria (EARS)
- When an L01/O14 PR reaction, an L02 monitor fire or a K01/K02/Q03 scheduled
  autorun fire is due for a task (or subgoal) that has a live terminal, the system shall deliver the trigger's
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
- When a task-level trigger is delivered and the task's own terminal (`task-<id>`
  or a session re-bound to the task) is live, the system shall deliver into that
  terminal, even if a subgoal terminal of the same task was active more recently.
- If a task-level trigger is delivered and no task terminal is live but a subgoal
  terminal of the task is, then the system shall deliver into the most recently
  active subgoal terminal and the message shall say it was routed there because
  the task terminal is not open.
- The system shall log, for every delivery, the chosen session and why it was
  chosen (subgoal's own / task's own / fallback to a subgoal terminal).
- The system shall start the helper by an absolute path to the agent CLI,
  resolved independently of how the app itself was launched (the app's PATH, the
  agent's standard install locations, then the user's interactive shell); if the
  helper cannot be started because the CLI is not found, the system shall
  re-resolve the path and retry once.
- When delivery fails, the fallback notification shall say the trigger was not
  delivered and name the session and the failure reason.
- The system shall not wake or poke an idle target session before sending: a
  peer message to an idle agent starts its next turn by itself.
- If the chosen terminal is hibernated (F23: tab kept, a plain shell, its
  conversation recorded), then the system shall wake it — resume that same
  conversation, exactly as viewing the tab would — wait until its agent is up,
  and then deliver; the trigger shall not be lost or turned into a notification
  only because its terminal was asleep.
- If the chosen terminal is neither hibernated nor running an agent and its
  foreground program is a plain shell, then the system shall start an agent in
  it before delivering: resume the terminal's own last known conversation (the
  status file's id for that terminal, else the F14 snapshot's id for that
  terminal — never another terminal's or the task's newest) when that
  conversation's transcript exists, wait until the agent is up and deliver; and
  otherwise start a fresh agent whose first prompt is the work-on-task /
  work-on-subgoal launch prompt followed by the delivered message.
- If a resumed conversation's agent does not come up, then the system shall
  start a fresh agent with the launch prompt and the delivered message instead.
- If the chosen terminal's foreground program is neither an agent nor a plain
  shell, then the system shall not type into it and shall fall back to the
  notification with the reason naming that program.
- When the system types an agent command into a terminal (a wake, a resume, a
  fresh start), it shall type the agent CLI's absolute path, so a shell whose
  PATH lacks the CLI's directory still starts it.
- If several peer sessions carry the target's terminal name, then the helper
  shall send to one of them — the most recently started — and treat that as a
  delivery, not a failure; if that one is unreachable it shall try the next.
- While a delivery to a session is in flight (from choosing the session to the
  helper's result), the system shall not hibernate that session (F23).
- If the helper finds no peer for the target and the target meanwhile lost its
  agent (it was put to sleep or exited), then the system shall make sure an
  agent is there again (wake / resume / fresh start, as above) and retry once.
- When a scheduled autorun fire is delivered into a live session (or falls back
  to the notification), the system shall still apply that fire's record change
  exactly as a launch would — `once` disarms, `cron` advances, `each` marks the
  occurrence fired — and shall release the autorun queue's gate at once, since no
  new session was started.
- The system shall deliver a scheduled autorun fire with the same work-on-task /
  work-on-subgoal prompt the fire would have typed into a newly opened terminal.

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
- **Scheduled fires** take the same path as external ones at the K01 commit
  point: the "terminal already open" branch no longer only notifies. The header
  names the mode (`scheduled autorun (once|cron|each): <title>`); the prompt is
  built by the same function the terminal launch uses, so the two can't drift.
  Delivery is fire-and-forget — the record patch and the gate release don't wait
  for the helper (up to ~150 s).
- This is the reference implementation's choice of transport. Any agent with a
  peer-messaging / inbox mechanism works the same way; without one, fall back to
  the notification.
- **Resolving the CLI.** Don't rely on `zsh -lc 'claude …'`: a login,
  non-interactive shell reads `.zprofile` but not `.zshrc`, and the native
  installer's `~/.local/bin` is commonly added only in `.zshrc`. The app then
  finds `claude` only when it happened to be started from an interactive
  terminal; started any other way, every delivery fails with
  `command not found: claude` (exit 127) until the next restart. Resolve once:
  the app's own `PATH`, then `~/.local/bin/claude`, `~/.claude/local/claude`,
  `/opt/homebrew/bin/claude`, `/usr/local/bin/claude`, then
  `$SHELL -ilc 'command -v claude'`; spawn the absolute path directly (no shell).
  Cache it, and drop the cache and retry once on `ENOENT`. The `~/.local/bin`
  symlink being swapped by an auto-update is harmless — the path stays valid.
- **Idle targets need no wake-up.** A peer message to an idle session is queued
  and starts its next turn by itself (observed: sessions idle 20 min – 8 h began
  a turn 2–5 s after the message arrived), so the helper only sends.
- **Asleep is not live.** A terminal restored or put to sleep by F23 still has
  its tmux session, so the session list alone calls it live, yet nothing in it
  can receive a peer message: the helper finds no peer, and a scheduled fire
  that took this path was silently lost (observed: two morning cron autoruns
  restored asleep after a restart). Check the hibernation record first: wake
  (type the resume), poll until the agent process is in the pane (~45 s cap),
  give it a few seconds to register as a peer, then send; if the helper still
  finds no peer, wait and retry once. Waking happens only for a trigger that is
  actually delivered — never to "check" a session.
- **Only a shell is the common case, not an edge.** In one week of logs 11 of
  ~100 live PR deliveries, plus a run of morning cron autoruns, failed with "no
  agent running (only a shell)": the agent had exited, or the terminal came back
  from a restart/restore without a hibernation record. Treat it like sleep:
  find the terminal's own last conversation id (status file → F14 snapshot entry
  for *that* terminal name; the task-wide history is ambiguous across a task's
  tabs), check its transcript exists under the account's `projects/`, type the
  resume, wait for the agent, deliver. No id or no transcript → type a fresh
  agent with the launch prompt + `[TaskMan] <header>` message as its first
  prompt (no helper needed — the prompt *is* the delivery). Check the pane's
  foreground command first: only `zsh`/`bash`/`sh`/`fish` (login `-zsh` too)
  counts as "only a shell".
- **Type the absolute CLI path.** Terminals restored long ago can carry a shell
  whose PATH lacks `~/.local/bin`; a typed `claude --resume <id>` then prints
  `command not found: claude` and the wake times out (observed: four morning
  autoruns). Type the same absolute path the helper is spawned with.
- **Race with F23.** A delivery can take a minute; F23's sweep can `/exit` the
  target in the middle of it (observed: an alert-driven task idle for hours was
  put to sleep between the readiness check and the helper's ListAgents, every
  ten minutes). Hold the session against hibernation for the whole delivery, and
  on a no-peer result re-check readiness and retry once.
- **Duplicate peer names.** Two peers can report the same tmux name (a stale
  registration next to the live one). Refusing as "ambiguous" lost the trigger;
  the helper sends to the most recently started one and, if it is unreachable,
  to the other.
- **Choosing among a task's sessions.** The lookup returns each live session with
  its kind (task's own / subgoal); a pure chooser picks task-own first for a
  task-level trigger. Ordering by "most recently active" across both kinds is
  the bug this rules out: a busy subgoal worker would steal every task-level
  trigger from the orchestrating task terminal.
- **Log each delivery** (`[deliver]`): target session, account, and outcome
  (`SENT <peer>` / no live session / failure reason), so a delivery can be
  audited after the fact — the success notification is gone by morning.
