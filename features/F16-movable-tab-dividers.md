---
id: F16
epic: F — In-app terminals
title: Movable tab dividers
size: S
requires: [F06, F07]
novel: false
---

## What
The user can drop one or more **dividers** into the terminal tab strip — thin
vertical separators that sit between tabs. A divider is a first-class draggable
item: it can be moved to any position and removed, exactly like a real tab is
dragged and closed. Dividers carry no session; they are a pure visual grouping
of the tabs, and their positions persist across restarts.

## Why it exists
With many terminals open, the strip is one undifferentiated row. The user wants
to carve it into regions they arrange by hand — the immediate use is a "left =
active, right = inactive" split, but a divider is deliberately dumb: it marks a
boundary, and the user drags tabs across it themselves. No auto-sorting, so tabs
never jump around under the cursor as agent state changes.

> *"I need dividers in tabs view so I can move it as well as real tabs. My
> immediate goal — split 'left' active tabs with 'right' inactive tabs."*

## Acceptance criteria (EARS)
- The user shall be able to add a divider to the tab strip.
- The user shall be able to drag a divider and drop it at a new position among
  the tabs, and the strip shall render it at the dropped position.
- The user shall be able to drag a terminal tab across a divider, and the tab
  shall land on the side it was dropped on (a divider never blocks a reorder).
- The user shall be able to remove a divider.
- When the divider positions change, the system shall persist them so they are
  restored on the next launch.
- The system shall not create, rename, or restart any tmux session when a
  divider is added, moved, or removed — dividers are view-only.
- While a divider is being dragged, the system shall give the same drop-position
  indication used for tabs.

## Build notes
- The tab list is derived live from tmux sessions every refresh, and only the
  session-name order rides `.terminals.json` (main rebuilds entries from live
  sessions and drops any name that is not a session). Dividers therefore need
  their own persistence — keep it local (a small localStorage list, like the
  legacy tab-order key), anchoring each divider to the name of the tab on its
  right (special "end" anchor when it trails the strip). A tab that disappears
  re-anchors its divider to the next surviving tab so no divider is orphaned.
- Render the strip from a **unified item list** — terminals interleaved with
  dividers in saved order — so the existing F07 drag machinery moves both kinds
  with one code path. Reuse `dragName`/`dragOver`; a divider just uses a sentinel
  drag id rather than a session name.
- A divider is not a tab: it is not selectable (no active state), has no LLM
  status icon, and is skipped by the Tab-key "next attention" cycle and by
  `activeTerminal`.
- Multiple dividers are allowed. Adding one drops it at the end (or next to the
  active tab); the user drags it where they want.
