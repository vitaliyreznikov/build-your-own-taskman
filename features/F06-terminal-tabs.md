---
id: F06
epic: F — In-app terminals
title: Terminal tabs
size: M
requires: [F04]
novel: false
---

## What
A tab strip for a task's terminals: one tab per terminal, click to switch the
active terminal, with controls to add a new terminal and close a tab. The strip
turns the set of sessions from F04 into a familiar, navigable UI.

## Why it exists
Multiple terminals per task (F04) are only usable if you can see and move between
them at a glance. Tabs are the surface where the parallel-supervision UX lands —
they're where per-terminal status icons appear (H-epic) and where "jump to the
next terminal that needs me" navigates.

## Acceptance criteria (EARS)
- The system shall display one tab per live terminal for the current task.
- When the user clicks a tab, the system shall make that terminal the active,
  visible one (attaching/repainting per F01).
- The user shall be able to create a new terminal (new tab) and close a terminal
  from its tab.
- When a terminal is killed or created, the system shall update the tab strip to
  match the current set of sessions.

## Build notes
- The tab strip renders the pty manager's per-task session set; switching tabs
  reuses the terminal host view, so apply F01's evict-stale-element and repaint
  handling when the active tab changes.
- Leave room in the tab for a per-terminal status icon (H-epic) and keep tab
  order explicit — F07 (drag reorder) and F08 (active-tab contrast) build
  directly on this strip.
