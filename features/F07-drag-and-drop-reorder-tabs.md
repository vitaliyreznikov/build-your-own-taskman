---
id: F07
epic: F — In-app terminals
title: Drag-and-drop reorder tabs
size: S
requires: [F06]
novel: false
---

## What
The user can reorder a task's terminal tabs by dragging them within the tab
strip. The chosen order is the order the tabs render in.

## Why it exists
When you're running several agents on one task, tab position becomes spatial
memory — you want the important one first, the scratch shell last. Drag-and-drop
is the direct, no-menu way to arrange that.

> *"In parallel build feature that allow reorder tabs in terminal — actually I
> want drag and drop."*

A long strip makes dragging a tab from one end to the other tedious, so the tab's
menu also jumps it straight to either end.

> *"to this menu I need move to the left and move to the right — to the first or
> to the latest position"*

## Acceptance criteria (EARS)
- The user shall be able to drag a terminal tab and drop it at a new position in
  the strip.
- When the user drops a tab, the system shall reorder the tabs to match and
  render them in the new order.
- The system shall not disturb the underlying tmux sessions or the active
  selection when only the order changes.
- While dragging, the system shall give a visual indication of the drop position.
- The tab's right-click menu shall offer **⇤ Move to first** and **Move to last ⇥**.
  When the user picks one, the system shall move that tab to the start / end of
  the app's tab order (the one shared order of F27, where built), so it shows first / last in **every**
  strip that holds it — not only the strip the menu was opened in.
- The system shall omit **Move to first** only on the tab that is already first
  in the shared order, and **Move to last** only on the tab already last in it —
  never because the tab happens to be first / last (or alone) in a filtered
  strip. A task panel's strip with a single tab therefore still offers both
  whenever other terminals are open elsewhere.
- Where intake tabs (F26) exist and lead the order, **Move to first** on any other
  tab shall place it right after them, so intake tabs keep the left end; on an
  intake tab it shall move to the very start.

## Build notes
- Reorder is a pure view-order concern — it must not rename or restart any
  session, so `TASKS_TERM_ID` and status keying (F05, H-epic) stay stable across
  a drag.
- "First / last" is measured in the shared order, not in the strip the user is
  looking at: the move is a reorder onto the shared order's first (non-intake) /
  last tab. Every other tab, hidden or shown, keeps its relative place. A divider
  anchored to the moved tab travels with it, as in a drag. (Measuring against the
  visible strip hid the item on a tab that was alone in a task panel, leaving no
  way to bring that terminal to the front of the other strips.)
- Reuse the board's existing drag-and-drop machinery if present rather than
  adding a second DnD library.
