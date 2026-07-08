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

## Acceptance criteria (EARS)
- The user shall be able to drag a terminal tab and drop it at a new position in
  the strip.
- When the user drops a tab, the system shall reorder the tabs to match and
  render them in the new order.
- The system shall not disturb the underlying tmux sessions or the active
  selection when only the order changes.
- While dragging, the system shall give a visual indication of the drop position.

## Build notes
- Reorder is a pure view-order concern — it must not rename or restart any
  session, so `TASKS_TERM_ID` and status keying (F05, H-epic) stay stable across
  a drag.
- Reuse the board's existing drag-and-drop machinery if present rather than
  adding a second DnD library.
