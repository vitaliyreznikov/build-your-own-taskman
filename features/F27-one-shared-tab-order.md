---
id: F27
epic: F — In-app terminals
title: One shared tab order for every strip
size: S
requires: [F07, F14]
novel: false
---

## What
There is one terminal tab order for the whole app. The main Terminal strip, the
strip filtered by category or ★ Starred, and a task panel's own Terminal strip
(and a subgoal's terminal list) all show the open terminals in that one order and
only differ in which tabs they hide. A reorder made in any of them — a drag, or
**Move to first / last** from the tab menu (F07) — shows in every other strip
that holds the tab.

## Why it exists
The task panel used to sort its tabs by itself (the task's own terminals by
index, then subgoal terminals), so a reorder made there didn't show, and a
reorder in the main strip didn't show in the panel.

> *"Moving a tab must change its position in every terminal view where that
> terminal appears — one shared ordering, not per-view."*

(An earlier version of this feature also added **← Move left / Move right →** to
the tab menu. That was a misreading of the ask — the user meant Move to first /
last, which F07 already provides — and the one-step items were removed.)

## Acceptance criteria (EARS)
- The system shall store one tab order for the app and shall render every
  terminal strip — the main strip under any category or ★ Starred filter, a task
  panel's own strip, and a subgoal's terminal list — in that order, hiding tabs
  it doesn't show.
- No strip shall sort the tabs it shows by any key of its own (task, subgoal,
  index); it shall only filter the shared order.
- When the tab order changes in one strip (drag, or F07 Move to first / last),
  the system shall show the new order in every other strip that contains the
  moved tab, without a restart.
- The system shall keep the order across an app restart and a reboot restore
  (F14).
- When a task's card jumps to "its" terminal, the system shall prefer the task's
  own terminal over a subgoal terminal that happens to sit further left.
- The tab right-click menu shall not offer one-step **Move left / Move right**
  items; F07's **Move to first / last** are the menu's reorder actions.

## Build notes
- The shared order is the renderer's single list of open terminals, already
  persisted with the terminal snapshot (F14). Views filter it; they must not
  sort it. A view that used to sort (task terminals first, then subgoal
  terminals, each by index) keeps that only as the order new tabs happen to be
  appended in.
- F07's Move to first / last reorder onto the first / last *visible* tab, so tabs
  hidden by a filter keep their places relative to each other.
