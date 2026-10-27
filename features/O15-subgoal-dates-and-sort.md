---
id: O15
epic: O — Structured (v2) task documents
title: Subgoal dates in the tree + sort by date
size: S
requires: [O10, K02]
novel: false
---

## What
Each subgoal row in the structured task view shows its **date**, the subgoal's
`nextAction` (the K02 "when it comes back" stamp), as a compact chip: `Oct 30
10:00`, with the year added when it isn't the current one. A past date on an
unfinished subgoal is shown as overdue. A **by date** toggle next to "open only"
reorders the tree by date: within every sibling list, subgoals that have a date
come first in ascending order, and undated ones follow in document order.

## Why it exists
A tax/admin task is mostly a list of things due at different moments; with ⏰
icons alone you have to hover each row to learn *when*. The user wants to see
the dates and read the list in calendar order.

> *"когда мы смотрим задачу v2 нужно показывать даты и дать возможность
> сортировки по датам"*

## Acceptance criteria (EARS)
- While a subgoal is not `done` and has a `nextAction`, the system shall show
  that date on its row, as `Mon D HH:mm`, adding the year when it isn't the
  current year.
- While that date is in the past, the system shall style the chip as overdue.
- The system shall not show a date chip on a `done` subgoal.
- When the user turns **by date** on, the system shall order every sibling list
  by sort key ascending, then undated subgoals in document order.
- A subgoal's sort key shall be its own date, else the earliest date among its
  unfinished descendants, so a parent sorts by the soonest work under it.
- When **by date** is off, the system shall show the document order.
- The system shall persist the toggle locally; sorting shall never rewrite the
  document or change any subgoal id.
- The sort shall compose with "open only" (O10): filter first, then sort.

## Build notes
- Purely presentational, like O10: sort a copy of the (filtered) tree.
- `nextAction` is a `YYYY-MM-DD HH:mm` local string, so a string compare orders
  it; only format it for display.
