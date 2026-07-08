---
id: H09
epic: H — LLM / agent status via Claude hooks
title: Jump to next terminal needing attention
size: M
requires: [H08]
novel: false
---

## What
A single keystroke jumps the user to the next terminal whose status is ⚠️
attention. Instead of scanning tabs, the human presses one key and is taken
straight to the next agent that needs them, cycling through all attention
terminals across tasks.

## Why it exists
This is parallel-supervision UX distilled to its essence. When you run dozens of
agents, the work is triage: find the one that needs you, deal with it, move to
the next. Making that a single keystroke turns the app from a board you watch
into a queue that routes you — the human becomes a review pipeline processing
attention events one keypress at a time.

> The author asked for exactly this: *"I want tab to bring me to the next tab
> needed my attention."* One keystroke to the next agent that needs him — pure
> parallel-supervision.

## Acceptance criteria (EARS)
- When the user triggers the jump, the system shall navigate to the next terminal
  whose status is attention.
- When multiple terminals need attention, the system shall cycle through them in
  order on repeated triggers.
- The system shall consider terminals across tasks, not only within the current
  task.
- When no terminal needs attention, the system shall do nothing (or clearly
  signal there is nothing to jump to) rather than navigate arbitrarily.

## Build notes
- Drive selection from the per-terminal status keying (H08); the jump target is
  any terminal whose keyed status is attention.
- Scope is "next needing attention" — attention only, matching the waiting-vs-
  attention semantics (H10). Do not include waiting terminals in the queue; the
  user is never "waited on."
- Bind to a keyboard shortcut in the terminal view; cycle deterministically so
  repeated presses walk the full set without skipping or repeating.
