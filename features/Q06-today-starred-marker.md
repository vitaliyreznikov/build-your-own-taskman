---
id: Q06
epic: Q — Time-bound tasks
title: Today shows ★ on rows whose task has a starred terminal
size: S
requires: [Q02, Q05, F22, O09]
novel: false
---

## What
A Today row gets a small **★** after its title when the work behind it has a
live terminal the user starred (F22): a task row when any of the task's open
terminals is starred (its own or a subgoal's), a subgoal row (Q05) when one of
that subgoal's own terminals is starred. The star reads as "this is in progress
right now". Hovering it lists the starred terminals. Rows without a starred
terminal show nothing, so the marker costs no space.

## Why it exists
Today is a list of deadlines; it didn't say which of them the user is already
working on. A starred terminal is the user's own "I'm on this" signal, so
surfacing it in Today separates started items from untouched ones at a glance.

> *"в списке today i want to see if we have starred terminal under the task —
> it's sort of sign 'in progress' situation"*

## Acceptance criteria (EARS)
- While a task has at least one open terminal that is starred (its own or one
  of its subgoals'), the system shall show ★ on that task's Today rows
  (current and held occurrences).
- While a subgoal has at least one open, starred terminal of its own, the
  system shall show ★ on that subgoal's Today row.
- The ★ shall carry a tooltip naming the starred terminals.
- A star on a terminal that is not open (a stale session name) shall not show.
- When a terminal is starred or unstarred, the Today marker shall update
  without a reload.
- Rows with no starred terminal shall show no marker.

## Build notes
- Derive it in the renderer from the F22 starred set and the live terminal
  list (with P05 re-binds applied); no new storage, IPC or file format.
