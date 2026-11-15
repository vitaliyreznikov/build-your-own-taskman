---
id: Q07
epic: Q — Time-bound tasks
title: "`each` autorun catch-up (auto-skip a stale occurrence so the next one fires)"
size: S
requires: [Q03, K01]
novel: false
---

## What
A repeating task (Q03) with **`Autorun = each`** launches its agent at the start of
every occurrence, once per occurrence. Q03 deliberately never advances a row on
its own, so if nobody marks an occurrence done or skipped, the row stays on it —
and because `each` already fired for that occurrence, **nothing ever launches
again**. The task is silently dead.

This feature carves out one exception to Q03's "nothing advances on its own",
for `each` tasks only:

- When the row's occurrence has **fully passed** (its Due is in the past) **and the
  next cron occurrence has already started**, the system closes the stale
  occurrence as `skipped`, marked as the system's doing:
  `{"id":"I185","due":"2026-10-28 12:05","state":"skipped","by":"auto","at":"2026-10-29 09:06"}`
- The row then advances **one occurrence per skipped line** — the same step a
  user's ✕ Skip takes (Q03). If several occurrences were missed (the app was closed
  for days), it catches up in **one pass**, one `skipped` line per missed
  occurrence, until it lands on the occurrence that is current now (or, if no
  occurrence is open now, the most recent one that has started).
- `each` then fires as usual — **once, for the occurrence the row lands on**. It
  never fires for the stale occurrences that were skipped on the way.

Plain repeating chores (Autorun off) keep Q03's behaviour: a passed occurrence stays
on the row, overdue in Today, until the user acts on it. That rule exists so the
user can tick yesterday's pills off late; an `each` task is different because
an agent, not the user, works each occurrence, and a stuck row stops the agent.

## Why it exists
Real case: a daily `cron:5 9 * * * +3h` task with `each`. The 2026-10-28 run fired,
nobody ticked it, and the row sat on 2026-10-28 09:05–12:05 for 18 days — no
launch, no notification beyond one overdue banner. An `each` task is "run this on
a schedule"; a schedule that stops after the first unacknowledged run is a bug.

## Acceptance criteria (EARS)
- While a task repeats, has `Autorun = each`, and is not DONE/CLOSED, when the row's
  Due has passed and the start of the next occurrence after the row's current one
  has arrived, the system shall append a `skipped` line with `"by":"auto"` for the
  row's occurrence (keyed by its scheduled Due, as Q03) and advance the row exactly
  one occurrence, as a user's skip does.
- The system shall repeat that step in the same pass while it still holds, so a row
  several occurrences behind lands on the latest started occurrence with one
  `skipped` line per occurrence passed over.
- The system shall not auto-skip an occurrence whose Due has not passed, even when
  the next occurrence has started (an overlapping window) — it is still open.
- The system shall not auto-skip occurrences of a repeating task whose Autorun is
  not `each`; Q03's "the row stays on the current occurrence" applies unchanged.
- When the row has been advanced this way, the `each` autorun shall fire once for
  the occurrence the row landed on, and shall not fire for any occurrence that was
  auto-skipped — including one that started while the app was closed and never
  fired.
- While an `each` task's row occurrence is stale (Due passed and the next
  occurrence started), the autorun scheduler shall not fire for it, so a launch
  that races the catch-up can't run the stale occurrence first.
- The system shall write the auto-skip lines by appending to `due-log.jsonl`
  (Q03); readers shall treat `"by":"auto"` `skipped` exactly like `skipped`.
- The system shall evaluate this whenever it evaluates due times (launch, any
  tasks.md change) and shall wake at the next occurrence's start of every `each`
  task, so the catch-up happens on time while the app runs.

## Build notes
- Reuse the skip step (Q03's "advance one occurrence"); don't write a second
  advance. The loop is the same shape as an extension that holds started
  occurrences (a follow-up task): walk `next start ≤ now`, emit one log line per
  step, write tasks.md once.
- If a task is both an `each` task and one whose open occurrences must be held
  (a follow-up extension), holding wins: the occurrence stays open for its reply
  and the row still advances, so `each` fires for the new one anyway.
- The "fired" record (`.autorun-each.json`) needs no change: the new occurrence has
  a new key, so it fires once; the guard against the stale occurrence lives in the
  autorun's target selection.
- Check passed-ness against the row's actual Due (which a hand edit may have moved),
  and the next start against the cron schedule.
