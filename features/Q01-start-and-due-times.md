---
id: Q01
epic: Q — Time-bound tasks
title: Start and Due times (rename NextAction → Start, add Due)
size: M
requires: [A02, B02, B10]
novel: false
---

## What
A task carries **two independent, optional times** instead of one:

- **Start** — *don't work on it before this time.* This is the old **NextAction**
  (B10), renamed: the `tasks.md` column becomes `Start` and every UI label says
  "Start". Its behaviour is unchanged — a task whose Start is in the future is
  hidden from boards by default (shown dimmed with "show hidden"), and autorun
  (K01) still fires at it.
- **Due** — *it must be done by this time.* A new `Due` column. A task with a Due
  is a **time-bound** task: it appears in the Today view (Q02), its card carries a
  due chip, and it turns overdue (red) once Due passes and it isn't done.

Both use the existing `YYYY-MM-DD HH:mm` local stamp. Either, both or neither may
be set: a big planned task usually has none (or only a deadline Due), a daily
chore has both (Start 08:00, Due 10:00).

## Why it exists
A personal board holds two very different kinds of work: big planned work that is
ranked by **priority**, and small things that must happen **at a time** (take
medicine, follow up on a payment). One time field could not say both "not before"
and "no later than": NextAction was used as a snooze for planned work and as a
deadline for chores, so a chore snoozed to its deadline looked identical to a
planned task snoozed for a week. Splitting the field lets one flag-free task model
carry both, and gives the Today view an unambiguous thing to sort by.

## Acceptance criteria (EARS)
- The system shall read each task's Start from the `Start` column of `tasks.md`;
  if the table has a `NextAction` column and no `Start` column, the system shall
  read that column as Start.
- When the system writes `tasks.md`, it shall write the header `Start` (never
  `NextAction`) in the position NextAction held, and shall write `Due` as a column
  after the existing ones, so readers that parse by position keep working.
- The system shall read each task's Due from the `Due` column; a row without that
  cell shall read as no Due.
- The UI shall label the not-before time "Start" everywhere it previously said
  "Next" / "Next action" (task panel, card tooltip, relations list).
- The task panel shall offer a Due picker next to the Start picker, with quick
  sets and a clear button.
- If a user edit would leave Start later than Due, then the system shall reject
  the edit and keep the stored values, and the UI shall say why.
- While a task is not DONE or CLOSED and has a Due, its card shall show a due chip
  (`⏱ <time>`); while that Due is in the past, the chip shall render as overdue.
- The board's existing Start-based hiding and sorting shall behave exactly as they
  did for NextAction.

## Build notes
- Back-compat is read-side only, as in A14: accept `NextAction`, always write
  `Start`. After one write the old header is gone.
- Keep the internal field name (`nextAction`) — v2 subgoal documents (K02) store
  the same concept under the JSON key `nextAction`, and renaming the code
  identifier without migrating those documents would split one concept across two
  names. The rename is of the column and the UI.
- Snooze (Q02) moves Start and Due together, so it keeps Start ≤ Due by
  construction. A repeating task's snooze moves only the current occurrence;
  its Start/Due stay read-only in the task panel (the schedule owns them).
- Update agent-facing docs (AGENTS.md, scheduling skill) in the same change:
  agents hand-edit `tasks.md` by column name.
