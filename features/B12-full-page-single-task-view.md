---
id: B12
epic: B — Kanban board UI
title: Full-page single-task view (level switcher)
size: M
requires: [B01, B06]
novel: false
---

## What
A "level switcher" that toggles between the board and a full-page view of a
single task, giving one task the whole window instead of a card and a side
panel.

## Why it exists
Once you're actually working a task — reading its note, and especially running
an agent in its terminal — you want the whole window on that one task, not a
column of cards. The full-page view is the workstation mode for a single task,
and it is the surface the in-app terminals (epic F) are hosted in.

## Acceptance criteria (EARS)
- When the user switches to the single-task level for a task, the system shall
  render that task full-page (its note and controls).
- When the user switches back, the system shall return to the board.
- The full-page view shall show the same task content as the note panel (B06).

## Build notes
- Renderer-level toggle between the board (B01) and a full-page single-task
  view; the full-page view reuses the note panel (B06) content.
- This view is the host for the embedded terminals (F01) — build it so a
  terminal surface can live alongside the note.
- The active view here is what the OS window title follows (B13).
