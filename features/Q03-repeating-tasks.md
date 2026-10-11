---
id: Q03
epic: Q — Time-bound tasks
title: Repeating time-bound tasks (Repeat cron + occurrence log)
size: M
requires: [Q01, Q02]
novel: false
---

## What
A new `tasks.md` column **`Repeat`**: empty (one-off) or
`cron:<m> <h> <dom> <mon> <dow>` optionally followed by a **window** `+<dur>`
(`+30m`, `+2h`, `+1d`, `+1h30m`). The cron is the same 5-field, local-time grammar
autorun uses (K01). Each cron occurrence is when an occurrence **starts**; its
**Due** is start + window (no window = due at the start time itself).
`cron:0 8,20 * * * +2h` = "08:00 and 20:00, each within two hours".

The row's Start/Due always hold the **current** occurrence. A repeating task never
moves to DONE by being done — it **advances**:

- **✓ Done** → log `done`, move Start/Due to the next occurrence.
- **✕ Skip** → same, logged `skipped`.
- **Missed** → when the *next* occurrence's start arrives while the current one is
  still open, the current one is logged `missed` and the task advances.

The current occurrence is always Due − window, so snoozing (which moves only
Start) never corrupts the schedule.

Every occurrence outcome is appended to **`due-log.jsonl`** at the data root, one
line each:
`{"id":"I601","due":"2026-10-11 10:00","state":"done","at":"2026-10-11 09:12"}`
(`state`: `done` | `skipped` | `missed`). Non-repeating tasks completed with a Due
are logged too.

## Why it exists
Daily chores (medicine twice a day, a weekly payment check) are one task that
recurs, not a new task per day. Keeping a single row keeps the board clean; the
occurrence log keeps the history (what was done, late, skipped, missed) that a
single Column value cannot hold.

## Acceptance criteria (EARS)
- The system shall read `Repeat` from `tasks.md`; a missing cell or empty value
  shall mean the task does not repeat. A value that is not a valid
  `cron:<expr> [+<dur>]` shall be treated as not repeating, and the task panel
  shall flag it.
- When a valid Repeat is set or changed on a task, the system shall set Start/Due
  to the occurrence that is open now (started, Due not yet passed) if there is one,
  else to the next occurrence after now.
- When a repeating task is marked done, the system shall append a `done` line to
  `due-log.jsonl`, set Start to the next cron occurrence after the current one and
  Due to that plus the window, and leave the task's Column unchanged.
- When a repeating task is skipped, the system shall do the same with `skipped`.
- When the next occurrence's start arrives while the current occurrence is not
  done or skipped, the system shall append a `missed` line for the current one
  and advance; if several occurrences were passed (e.g. the app was closed), it
  shall log each, up to a cap, and land on the newest occurrence that has started.
- When any task that has a Due is moved to DONE, the system shall append a `done`
  line for it.
- The system shall append to `due-log.jsonl`, never rewrite it.
- While a task repeats, its Autorun shall be either off or **`each`** — launch
  the task's agent terminal at the start of **every** occurrence (K01's countdown,
  gate and "terminal already open" rules apply). A `once`/`cron:` autorun on a
  repeating task shall not fire (the Repeat schedule is the one owner of Start).
- When Repeat is set on a task whose Autorun is `once` or `cron:`, the system
  shall change its Autorun to `each`, keeping the "an agent runs this" intent.
- The system shall fire an `each` autorun at most once per occurrence (keyed by
  the occurrence's Due), including across app restarts; an occurrence that
  started while the app was closed and is still open shall fire once at launch.
- An `each` autorun shall not advance or clear anything — done / skip / missed
  (above) move the task to the next occurrence, and that occurrence fires again.
- The UI shall mark repeating tasks with 🔁 on cards and Today rows.

## Build notes
- The window lives in the Repeat cell, not in the Start/Due gap: Start is
  user-movable (snooze), so a lead derived from it would drift.
- Share the cron parser and "next occurrence" helper with K01 if you built it.
- The missed check runs in the main process on app start, on every `tasks.md`
  change, and on a timer armed for the earliest next-occurrence start.
- `each` exists so two schedulers never both advance Start: Repeat owns the
  times, autorun only reacts to them. Record "fired for occurrence X" in a
  machine-local file (it describes what this machine launched, like tmux state),
  not in tasks.md.
