---
id: F08
epic: F — In-app terminals
title: High-contrast active tab
size: S
requires: [F06]
novel: false
---

## What
The active terminal tab is styled with clearly higher contrast than the inactive
ones, so which terminal is currently in view is obvious at a glance.

## Why it exists
With several tabs — each possibly showing a status icon — the eye needs an
instant answer to "which one am I looking at?" The default styling made active
and inactive tabs too similar to tell apart quickly.

> *"On terminal tab let's show with more contrast which tab is active — now it's
> hard to distinguish that."*

## Acceptance criteria (EARS)
- The system shall render the active terminal tab with visibly higher contrast
  than inactive tabs.
- When the active terminal changes (click, reorder, jump-to-attention), the
  system shall move the high-contrast styling to the new active tab.
- The styling shall remain legible alongside a per-terminal status icon and in
  both light and dark themes.

## Build notes
- Purely presentational — a styling change on the F06 tab strip; no session or
  state changes.
- Verify contrast against the status-icon colors so the active-tab emphasis and
  the icon don't wash each other out.
