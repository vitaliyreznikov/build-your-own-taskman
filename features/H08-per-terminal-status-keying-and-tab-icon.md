---
id: H08
epic: H — LLM / agent status via Claude hooks
title: Per-terminal status keying and tab icon
size: M
requires: [F05, H03]
novel: false
---

## What
Status is keyed per terminal, not just per task. Each terminal exports its own
id (`TASKS_TERM_ID`, the tmux session name), so a task running several agents in
several terminals gets one status per terminal — and each terminal **tab** shows
its own live status icon. The tab strip becomes a row of live agent states.

## Why it exists
A single task often has multiple agents working in parallel terminals. A
task-level status would collapse them into one misleading icon. Keying per
terminal, and surfacing the icon on each tab, lets the human see at a glance which
of several agents on the same task is working, waiting, or needs them — the
parallel-supervision idea pushed down to the terminal level.

> Terminal View, in the author's words: *"In Terminal View I need to see LLM State
> in tab names."* Seeing state per tab is what makes juggling many agents on one
> task legible.

## Acceptance criteria (EARS)
- The system shall key agent status on `TASKS_TERM_ID` so each terminal has its
  own status entry.
- When a terminal has a status, the system shall render the corresponding icon on
  that terminal's tab.
- The system shall show per-terminal status icons in both the task's tab strip
  and the full-screen terminal tab strip.
- When a task has multiple terminals, the system shall not collapse their
  statuses into a single task-level icon.

## Build notes
- `TASKS_TERM_ID` is exported by the ptyManager per terminal (F05, multi-terminal
  + per-terminal id export); the hooks (H03) must key the status file (H02) on it.
- The card-level icon (H01) can still summarize a task; the per-terminal icons
  live on the tabs. Keep both fed from the same keyed status entries.
- Render the icon in the tab strip and the full-screen tab strip — both surfaces
  the author called out.
- This keying is the prerequisite for jump-to-next-attention (H09), which needs
  to know which specific terminal needs the user.
