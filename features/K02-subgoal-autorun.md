---
id: K02
epic: K — Scheduled autorun
title: Scheduled autorun (once + cron) of a single subgoal's Claude terminal
size: M
requires: [K01, O03, O09]
novel: false
---

## What
K01 arms a **task** to launch itself at its `NextAction`. K02 pushes the same
mechanism down one level: a **subgoal** of a v2 document (O03) can be armed to
launch **its own** terminal (O09) when its scheduled moment arrives — no click,
no human present. It is K01's scheduler generalized from "task" to "task *or*
subgoal," reusing the same cancelable countdown, the same one-at-a-time queue,
and the same post-fire disarm/advance rules.

Two new per-subgoal fields carry the schedule, stored inside the subgoal node in
`I<N>.json` (never in `tasks.md`, where subgoals do not live):

- **`autorun`** — the same grammar as a task's: `''` (off), `once`, or
  `cron:<m> <h> <dom> <mon> <dow>` (a raw 5-field crontab expression in local
  time).
- **`nextAction`** — a `YYYY-MM-DD HH:mm` local stamp; the single source of truth
  for *when*, exactly as it is for a task.

Firing opens the **subgoal's** terminal (`task-<id>-sg-<slug>`, O09), auto-starts
the agent, and briefs it on **that one subgoal** (the O09 work-on-subgoal prompt),
not the whole task — a v2 document routinely has dozens of subgoals, so
"work the task" is the wrong instruction when a single subgoal came due.

`once` fires a single time then **disarms** (clears the subgoal's `autorun` and
`nextAction`); `cron` fires then **stays armed** and advances the subgoal's
`nextAction` to the next occurrence. A subgoal that is `done`, or whose parent
task is in a terminal/closed column (DONE/CLOSED), never fires.

### One shared scheduler, one shared queue
K02 does **not** add a second scheduler. Task autorun and subgoal autorun share
K01's single serialized queue and its post-fire gate, so a batch of things coming
due at once still launches **one terminal at a time** (waiting for each fired
session to settle) rather than spawning a storm of terminals that fight for focus.
The gate keys on the fired session's llm-status key — `task-<id>` for a task,
`task-<id>-sg-<slug>` for a subgoal.

### Arming is driven off document changes
A task's schedule is re-armed when `tasks.md` changes; a subgoal's schedule is
re-armed when its `I<N>.json` changes — the app already watches those documents
(O07) and re-derives their index entry, so the armed set is read from that
in-memory index without a per-file disk scan. Setting/editing a subgoal's schedule,
and the app's own post-fire write-back, are document changes and therefore
re-arm naturally.

## Why it exists
O09 gave a subgoal its own terminal so an agent can be pointed at one step of a
large task; K01 removed the friction of *being present at the right time* for a
task. K02 is the intersection: recurring or deferred work that belongs to **one
step of a decomposed task** — "re-check this PR's rollout every morning," "resume
this migration phase after the maintenance window," "poll this vendor daily until
they reply" — becomes "arm the subgoal once and walk away," while the rest of the
document stays untouched. Without it, the only schedulable unit is the whole task,
which is too coarse for a v2 document where different subgoals move on different
clocks.

> The reference author runs many long v2 tasks whose subgoals each wait on a
> different external clock (a CI run, a warm-up ramp, a person). Arming the *task*
> would re-brief the agent on the entire document every morning; arming the
> *subgoal* points it at exactly the step that came due.

## Acceptance criteria (EARS)
- The system shall let a subgoal of a v2 document carry an `autorun` value of
  `once` or `cron:<expr>` plus a `nextAction` stamp, persisted in the subgoal node
  inside the git-synced `I<N>.json`.
- When an armed subgoal's `nextAction` time arrives while the app is running, the
  system shall begin a fire for that subgoal.
- When a fire begins, the system shall show a cancelable countdown (notification +
  in-app banner identifying the subgoal) before launching the agent.
- When the countdown elapses without cancellation, the system shall open **that
  subgoal's** terminal (O09), move the parent task to IN PROGRESS (F09),
  auto-start the agent, and type the **work-on-subgoal** prompt.
- When a `once` subgoal fire completes, the system shall clear that subgoal's
  `autorun` and `nextAction`.
- When a `cron` subgoal fire completes, the system shall keep the subgoal's
  `autorun` and set its `nextAction` to the next occurrence of the expression.
- When the user cancels a `once` subgoal fire, the system shall disarm that
  subgoal; when the user cancels a `cron` subgoal fire, the system shall skip that
  occurrence and advance the subgoal's `nextAction` to the next.
- The system shall not fire a subgoal that is `done`, nor any subgoal whose parent
  task is in a terminal/closed column (DONE/CLOSED).
- The system shall serialize task and subgoal fires through one shared queue so
  that multiple items coming due at once launch one terminal at a time.
- When a subgoal fire finds a live terminal already open for that subgoal, the
  system shall surface a notification instead of launching a second agent on top
  of it.
- When a subgoal's `cron` expression has no computable next occurrence (malformed),
  the system shall treat that subgoal as un-armed rather than fire.

## Build notes
- Reuse, don't fork: generalize K01's scheduler to a target that is a task **or** a
  subgoal (`{ taskId, subgoalId? }`), keyed by a composite id (`I166` vs
  `I166#g1.2`). Keep the one queue, the one gate, the one 15-second countdown.
- Enumerate armed subgoals from the already-maintained document index (O07), not by
  re-reading every `I<N>.json` from disk. Skip `done` subgoals and subgoals whose
  parent task is DONE/CLOSED.
- The fire path is the existing O09 `openSubgoalTerminal(taskId, subgoalId)` +
  work-on-subgoal prompt — the same route O07 already uses to resume a subgoal on
  a returned PR review — so IN PROGRESS membership, agent start, and the status
  prompt all come for free. The only new main→renderer detail is a `subgoalId`
  (and its text) on the fire signal.
- The post-fire disarm/advance writes back into the subgoal node in `I<N>.json`
  (read doc → patch the node → write doc), as an own-write, then re-arms once.
- Compute the next `cron` occurrence and write it into the subgoal's `nextAction`
  both when the expression is set/edited and after each fire, mirroring K01.
- UI: a per-subgoal autorun control (off / once / cron-with-expression) and a
  "next" time picker in the structured task view, plus a small ⏰/🔁 badge on the
  subgoal row — the subgoal-scoped twin of K01's task-detail control and card
  badge.
- The `autorun`/`nextAction` fields are git-synced (subgoal configuration); the
  transient "a countdown is showing" state stays machine-local, like `.llm-status`.
