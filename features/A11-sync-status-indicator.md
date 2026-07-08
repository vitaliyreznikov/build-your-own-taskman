---
id: A11
epic: A — Markdown data model
title: Sync status indicator
size: S
requires: [A09]
novel: false
---

## What
A small UI element (`SyncStatus`) that shows the state of the background sync:
when the last sync completed, whether a sync is in progress, or whether the last
attempt errored. It also offers a manual "sync now" action to trigger a commit/
mirror on demand.

## Why it exists
Sync is automatic and invisible (A09/A10), which is good until something goes
wrong — then the human needs a way to know their files are (or aren't) safely
committed, and a way to force a sync before, say, closing the laptop. The
indicator turns the silent background engine into something trustworthy at a
glance.

## Acceptance criteria (EARS)
- The system shall display the time or relative age of the last successful sync.
- While a sync is running, the system shall show an in-progress state.
- When a sync fails, the system shall show an error state distinct from the idle
  and in-progress states.
- When the user activates "sync now", the system shall trigger an immediate
  sync and reflect its progress and outcome.

## Build notes
- Lives in the renderer; reads state reported by the sync engine (A09) over IPC —
  it's a display of the engine's status, not its own sync logic.
- Cover all three outcomes distinctly (last-sync / in-progress / error); an error
  should be visible, not swallowed, since it means files may be uncommitted.
- "Sync now" is a manual override of the debounce/periodic schedule for when the
  human wants certainty right now.
