---
id: K01
epic: K — Scheduled autorun
title: Scheduled autorun (once + cron) of a task's Claude terminal
size: L
requires: [E01, E03, F09, F12]
novel: true
---

## What
A task can be **armed to launch itself**. When its `NextAction` time arrives, the
app automatically opens the task's terminal and auto-starts the agent (exactly the
F12 "open terminal → `claude` → work-on-task prompt" ritual, plus the F09 move to
IN PROGRESS) — no click required. It is the timer half of E01 (which today only
*notifies* at `NextAction`) wired to the terminal half of F12, so a scheduled
moment doesn't just remind you, it *does the thing*.

Two flavors, carried in a new `autorun` field on the task:

- **`once`** — fire a single time at `NextAction`, then **disarm**: clear the
  `autorun` field and clear `NextAction`. ("The label should be removed.")
- **`cron:<m> <h> <dom> <mon> <dow>`** — a raw 5-field crontab expression
  (evaluated in **local time**). Fire at `NextAction`, then **stay armed** and
  advance `NextAction` to the next occurrence of the expression.

`NextAction` remains the single source of truth for *when*: for `once` it is the
time you set; for `cron` the app **computes the next occurrence and writes it into
`NextAction`** whenever the expression is set/edited and after every fire. So the
existing "future `NextAction` ⇒ dimmed/hidden" behaviour (B10) and the existing
long-delay timer (E03) keep working unchanged — autorun is purely an extra action
layered onto the moment the timer fires.

### Guardrail: cancelable countdown
Because a fire launches an *autonomous* agent, firing is not instantaneous. On
fire the app shows a **~15-second cancelable countdown** (a desktop notification +
an in-app banner) before it types `claude`:

- Cancel a `once` fire → **disarm it** (clear `autorun`); the user chose not to run
  it this time.
- Cancel a `cron` fire → **skip this occurrence** and advance `NextAction` to the
  next slot.
- Closing the app during a countdown counts as a skip; the fire is re-evaluated as
  overdue on next launch (see catch-up).

### Catch-up on launch (the app must be running to launch a terminal)
Launching a terminal is a renderer/xterm operation, so autorun can only fire while
the desktop app is open. If an armed task's `NextAction` is already in the past
when the app starts, it fires **once** immediately (through the same countdown).
Several overdue tasks fire **sequentially**, not all at once, so the user doesn't
get a wall of terminals. A `cron` task then advances to its next future slot.

## Why it exists
The whole app is "point an agent at a task." F12 removed the friction of typing
`claude` + the boilerplate; autorun removes the friction of *being present at the
right time*. Recurring chores (a nightly sync, a Monday-morning triage, a
usage-window warmup) become "arm it once and walk away" instead of "remember to
open the terminal." It turns `NextAction` from a passive radar pointer into an
active scheduler while keeping the human in the loop via the countdown.

> The reference author already runs a daily headless warmup via an OS LaunchAgent;
> autorun brings that pattern *inside* TaskMan, where it can target any task, carry
> the task's note file, and surface in the same board as everything else.

## Acceptance criteria (EARS)
- The system shall let a task carry an `autorun` value of `once` or a
  `cron:<expr>` recurrence, persisted in the git-synced task index.
- When an armed task's `NextAction` time arrives while the app is running, the
  system shall begin a fire for that task.
- When a fire begins, the system shall show a cancelable countdown (notification +
  in-app banner) before launching the agent.
- When the countdown elapses without cancellation, the system shall open the
  task's terminal, move it to IN PROGRESS (F09), auto-start the agent, and type the
  work-on-task prompt (F12).
- When a `once` fire completes, the system shall clear the task's `autorun` field
  and clear its `NextAction`.
- When a `cron` fire completes, the system shall keep the `autorun` field and set
  `NextAction` to the next occurrence of the expression.
- When the user cancels a `once` fire, the system shall disarm it; when the user
  cancels a `cron` fire, the system shall skip that occurrence and advance to the
  next.
- When an armed task's `NextAction` is already past at app launch, the system shall
  fire it once (with countdown), firing multiple overdue tasks sequentially.
- Autorun shall arm regardless of task priority (unlike the P0-only notification of
  E01) and shall not fire for tasks in a terminal/closed column (DONE/CLOSED).
- When a `cron` expression has no computable next occurrence (malformed), the
  system shall treat the task as un-armed and surface it as invalid rather than
  fire.

## Build notes
- Reuse, don't fork: the fire action is the existing `termOpen({ taskId,
  autoClaude:true })` path so IN PROGRESS membership (F09), agent start, and the
  status-marker prompt (F12) all come for free. The only new main→renderer signal
  is "please open this task's terminal now."
- Arming is independent of the E01 notification qualifier (which is P0-only). Give
  autorun its own scheduling pass over the task list so a P2 chore still fires; it
  can share E03's long-delay chunking for `NextAction` values far in the future.
- Cron: a small local-time 5-field evaluator (min hour dom mon dow). Compute
  "next occurrence strictly after *now*" both when the expression is set and after
  each fire, and write it into `NextAction`. Do not attempt to catch up *every*
  missed cron slot — collapse a backlog to a single catch-up fire, then jump to the
  next future slot.
- The countdown is the safety valve for an autonomous launch — keep it visible and
  trivially cancelable (matches the project's "no unguarded auto-apply of sensitive
  actions" stance). Batch/serialize catch-up fires so overdue tasks don't spawn a
  storm of terminals.
- UI: a card badge (⏰ once / 🔁 cron) and an autorun control in the task's detail
  panel next to the `NextAction` picker (off / once / cron-with-expression).
- The `autorun` field is git-synced (it's task configuration), but the transient
  "a countdown is currently showing" state is machine-local, like `.llm-status`.
