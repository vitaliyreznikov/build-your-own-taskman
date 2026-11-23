---
id: H14
epic: H — LLM / agent status via Claude hooks
title: Break nudge when no agent needs you (external focus timer + busy-strip rule)
size: M
requires: [H08, H10, F23, F25, F27, E01]
novel: true
---

## What
An external focus timer (any Pomodoro-style app the user already runs) publishes
its state to a small local file. TaskMan shows the time since the last break as a
**focus chip at the left end of the terminal tab strip**, and once the user has
been focusing for 20 minutes it suggests a **short break at the moment the break
costs nothing**: when every agent near the front of the strip is either working or
parked on something external, so none of them is waiting for the user.

The contract between the two programs is two gitignored files at the data root:

- `.focus-state.json` — written by the focus timer, read by TaskMan:
  `{"v":1,"source":"<app>","phase":"focusing"|"break"|"idle","focusElapsedSec":1234,"breakEndsAt":"<ISO>"|null,"updatedAt":"<ISO>"}`.
  `focusElapsedSec` is the timer's own "time accrued toward the next break" — the
  timer decides what resets it (a finished break, a stretch of inactivity). The
  timer rewrites the file on every phase change and at least every 30 s.
- `.focus-command.json` — written by TaskMan, consumed by the focus timer:
  `{"v":1,"cmd":"break"|"prompt","id":"<unique>","at":"<ISO>"}`. `break`: the timer
  starts its own normal break (its own length, its own overlay, its own logging).
  `prompt`: the timer raises its own "time for a break" prompt — TaskMan has found
  a quiet moment. Either way the timer deletes the file.
- `.focus-log.jsonl` — TaskMan's own append-only log of nudge events (nudge
  posted, prompt/break command written, explanation shown, the user's answer,
  nudge closed and why), one JSON object per line, so a "why did/didn't it fire"
  question can be answered after the fact. The timer logs its side in its own log.

The focus timer **does not prompt on its own clock**: its break prompt appears
only when TaskMan asks for it. Its blocking break and its inactivity reset stay.

The **busy-strip rule** looks at the terminal tabs **currently shown in the strip**
(after the category / ★ filter, F20/F22), in the shared tab order (F27), left to
right — the user judges by what they see, so a hidden tab never blocks or allows a
break. It skips every tab that tells nothing about whether the user is
needed:

- tabs marked as orchestrator (F25) — a coordinator is usually idle by design;
- tabs with no agent status (a plain shell) and hibernated tabs (F23);
- tabs in `done` whose status is older than 10 minutes — a finished task left
  open, not an agent waiting for its next instruction.

Of the remaining tabs it takes the **5 leftmost**. The strip is *busy* when there
are at least 2 of them and every one is `working` or `waiting` (H10: waiting is
never waiting on the user). Any `attention`, `approval` or recent `done` among
them means someone needs the user — answer them first.

## Why it exists
A fixed-interval break reminder fires at a random moment: mid-review, or while
three agents sit idle waiting for an answer, so the break stalls all of them and
gets skipped. With several agents working in parallel the right moment is
visible: when nobody waits for you, a few minutes away costs zero throughput.
The focus timer already knows how long the user has been working; TaskMan knows
whether anyone is waiting. Neither can decide alone.

> *"After 20 min of work, TaskMan shows a timer; when the 5 leftmost terminals are
> all busy — nobody waits for me — it proposes a short break."*

## Acceptance criteria (EARS)
- While `.focus-state.json` exists, has `updatedAt` within the last 3 minutes,
  `phase` is `focusing` and the elapsed focus time (`focusElapsedSec` plus the time
  since `updatedAt`) is at least 20 minutes, the system shall show a focus chip at
  the left end of the main terminal tab strip with that time in whole minutes.
- While the elapsed focus time is under 20 minutes, the system shall show no chip
  at all — no progress is displayed before a break is due.
- The system shall refresh the chip at most every 30 seconds (no seconds ticking).
- While `phase` is `break`, the system shall show the chip as a break with the
  time remaining until `breakEndsAt`.
- If the file is missing, unreadable, older than 3 minutes, or `phase` is `idle`,
  then the system shall hide the chip and never nudge.
- The system shall show the focus chip in an amber "break due" style.
- While the elapsed focus time is at least 20 minutes and the busy-strip rule
  holds, the system shall nudge: the chip pulses, a desktop notification offers
  **Break** and **Later**, and the system writes `.focus-command.json` with
  `cmd: "prompt"` and a fresh `id` so the focus timer raises its own break prompt.
- While a nudge is up, the system shall keep the recommendation on screen and
  current: every minute it shall replace the notification (same one, not a new
  stack entry) with the updated focus time ("Focused 23 min — break
  recommended"), silently after the first post, whether or not the busy rule
  still holds, until the user acts on it or the cycle ends.
- The system shall start at most one nudge per focus cycle; a cycle ends when the
  elapsed time drops below 20 minutes (the timer reset it or a break started), and
  then the system shall remove the notification.
- When the user picks **Later**, the system shall nudge again no earlier than 10
  minutes later, under the same rule.
- When the user picks **Break** (in the notification or on the chip), the system
  shall write `.focus-command.json` with `cmd: "break"` and a fresh `id`, and stop
  nudging.
- When the user picks **Already rested** on the chip, the system shall restart the
  elapsed count from that moment for display and nudging, without commanding the
  focus timer.
- When computing the busy-strip rule, the system shall skip orchestrator-marked,
  status-less and hibernated tabs and `done` tabs older than 10 minutes, take the
  5 leftmost remaining tabs among those currently shown in the strip (after the
  category/starred filter), in the shared tab order, and hold only when there are
  at least 2 and all are `working` or `waiting`.
- While the elapsed focus time is at least 30 minutes and the system has not
  nudged in this cycle (the busy rule never held), the system shall show next to
  the chip a short explanation of why no break was suggested — which shown tabs
  need the user (e.g. "no quiet moment: I2 ⚠️, I813 ⚠️") — together with
  **Break** and **Later** buttons, so the user can take the break anyway.
  This adds no notification and sends no `prompt` command.
- The focus timer, on finding a `.focus-command.json` with `cmd: "break"` while
  focusing, shall start its own normal break and delete the file; with
  `cmd: "prompt"` while focusing it shall raise its own break prompt (unless one
  is already up); on any other phase it shall delete the file without acting.
- The focus timer shall not raise its break prompt on its own focus-duration
  clock; its blocking break and its inactivity reset keep working unchanged.
- The system shall append a line to `.focus-log.jsonl` (`at`, `event`, elapsed
  minutes, and for a nudge or explanation the tabs checked) when it posts a
  nudge, writes a command, first shows the explanation in a cycle, receives the
  user's answer, and closes a nudge; the focus timer shall log each command it
  receives and each prompt or break it starts.

## Build notes
- All three files are machine-local: add them to the data root's `.gitignore`.
- Main process: watch `.focus-state.json` (fs.watch + a 30 s poll as backstop),
  parse defensively, push `{phase, focusElapsedSec, breakEndsAt, updatedAt}` to the
  renderer over IPC; expose `focusBreakNow()` that writes `.focus-command.json`
  atomically (tmp + rename).
- Renderer: the chip is the strip's first element; a 30 s tick derives elapsed time.
  The busy rule is a pure function of (tabs in shared order, status map,
  orchestrator set, hibernation records, now) — unit-test it.
- The nudge notification goes through the main process like other actionable
  notifications (E01); keep the "nudged for this cycle" / "snoozed until" state in
  memory — a restart may nudge again, which is harmless.
- "Replace" on macOS = close the previous `Notification` object and show a new
  one: Notification Center then holds only the latest. Re-posts are `silent`.
  Native "Alerts" style is not required — the per-minute re-post is what keeps
  the recommendation visible.
- Reference timer: the author's macOS app writes the state from its focus state
  machine tick (throttled to phase changes + every 30 s) and checks for the command
  file every few seconds. Its own 30-minute prompt was removed: with the user in
  TaskMan all day it fired at random moments; the TaskMan-timed one fires when
  nobody waits.
