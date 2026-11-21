---
id: F27
epic: F — In-app terminals
title: Move a tab one step left / right, in one shared order
size: S
requires: [F07, F16, F22, F23, F26]
novel: false
---

## What
A terminal tab's right-click menu gains **← Move left** and **Move right →**.
Each moves the tab one step in the strip the user is looking at, so the tab can
be nudged into place without dragging.

There is one tab order for the whole app. The main Terminal strip, the strip
filtered by category or ★ Starred, and a task panel's own Terminal strip all
show the open terminals in that one order and only differ in which tabs they
hide. A move made in any of them shows in every other one that holds the tab.

## Why it exists
Dragging a tab is awkward on a long strip and impossible to do precisely when the
target is one slot away. The task panel also used to sort its tabs by itself
(the task's own terminals by index, then subgoal terminals), so a reorder made
there didn't show, and a reorder in the main strip didn't show in the panel.

> *"The terminal tab right-click menu needs a 'Move left' option. Moving a tab
> left must change its position in every terminal view where that terminal
> appears — one shared ordering, not per-view."*

## Acceptance criteria (EARS)
- When the user right-clicks a terminal tab, the system shall show
  **← Move left** and **Move right →** in the menu.
- When the user picks **Move left** on a tab, the system shall move the tab one
  step to the left in the strip as currently shown; picking **Move right** shall
  move it one step to the right.
- The system shall disable **Move left** on the leftmost tab of the strip and
  **Move right** on the rightmost tab, counting dividers (F16) as steps.
- When the step to the left or right is another tab, the system shall swap the
  two tabs in the shared order and leave every tab the current strip hides
  where it is relative to the other hidden tabs.
- When the step to the left or right is a divider (F16), the system shall move
  the tab across that one divider into the neighbouring group and shall not
  change the tab order.
- When a tab is swapped with a neighbour tab, the system shall keep each divider
  between the same groups: a divider that sat just left of the pair stays just
  left of it.
- The system shall store one tab order for the app and shall render every
  terminal strip — the main strip under any category or ★ Starred filter, and a
  task panel's own strip — in that order, hiding tabs it doesn't show.
- When the tab order changes in one strip, the system shall show the new order
  in every other strip that contains the moved tab, without a restart.
- The system shall keep the moved order across an app restart and a reboot
  restore (F14).
- The system shall move an asleep tab (F23) without waking it.
- Moving a tab shall not change which tab is active, rename or restart its
  session, or change its star (F22), orchestrator mark or account.
- When a new intake tab opens, the system shall still put it at the left end of
  the strip (F26); after that it moves like any other tab.

## Build notes
- The shared order is the renderer's single list of open terminals, already
  persisted with the terminal snapshot (F14). Views filter it; they must not
  sort it. A view that used to sort (task terminals first, then subgoal
  terminals, each by index) keeps that only as the order new tabs happen to be
  appended in.
- Implement the step as a pure function over (full order, shown order, dividers,
  tab, direction) → (new order, new dividers) so it can be unit-tested without
  the UI. A task panel passes no dividers, so it only ever swaps tabs; the
  divider re-anchoring still runs on the shared divider list so the main strip
  stays consistent.
- Swap left onto tab Y: put the tab just before Y; dividers anchored to Y
  re-anchor to the moved tab; dividers anchored to the moved tab re-anchor to its
  old successor. Swap right onto tab N: put the tab just after N; dividers
  anchored to the moved tab re-anchor to its old successor.
- Cross a divider leftward: re-anchor that divider to the next tab shown to the
  right of the moved tab (or to the strip end when there is none). Cross a
  divider rightward: re-anchor that divider to the moved tab.
- **Move to first / last** (F07) stay as they are and now also land in every
  view, because the task panel no longer re-sorts.
