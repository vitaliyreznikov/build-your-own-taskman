---
id: F11
epic: F — In-app terminals
title: Terminal-state tally in sidebar
size: S
requires: [F04, H01]
novel: false
---

## What
A small snippet in the sidebar (under the search box) that aggregates all live
terminals by their current state and shows how many are in each — e.g. how many
are working, waiting, or need attention.

## Why it exists
When you're running many agents across many tasks, you want a single
at-a-glance count of the fleet: is anything waiting on me right now? The tally
is the top-level readout of the parallel-supervision system — it summarizes the
per-terminal status (H-epic) into one glanceable line.

> *"add small snippet on terminal states — it should show how many terminal per
> state we have."*

## Acceptance criteria (EARS)
- The system shall display, in the sidebar, a count of live terminals grouped by
  their current LLM/agent state.
- When any terminal's state changes, the system shall update the tally to match.
- When terminals are created or killed, the system shall include/exclude them
  from the counts accordingly.

## Build notes
- Read from the same transient status source the H-epic maintains, keyed by
  `TASKS_TERM_ID` (F05); this is a pure aggregation view, not a new source of
  truth.
- Keep it quiet: no blinking or attention-grabbing animation — it's a summary
  readout, consistent with the "no icons blinking" principle.
