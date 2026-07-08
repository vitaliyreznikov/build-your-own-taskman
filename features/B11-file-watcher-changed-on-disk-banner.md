---
id: B11
epic: B — Kanban board UI
title: File watcher and changed-on-disk banner
size: M
requires: [A01]
novel: false
---

## What
A watcher over the data files that detects when they change on disk outside the
app, and an in-app banner (`UpdateBanner`) prompting the user to refresh /
reload the new content.

## Why it exists
The files are the source of truth and other things edit them — an agent working
a task, an external editor, the git/Obsidian sync. The app must never let its
in-memory view silently win over the file. The watcher-plus-banner is what makes
external edits Just Work: the app notices the file moved out from under it and
offers to catch up.

## Acceptance criteria (EARS)
- When a data file changes on disk outside the app, the system shall detect the
  change.
- When an external change is detected, the system shall show a banner offering
  to refresh to the new content.
- When the user accepts the refresh, the system shall re-read the changed files
  and update the view.
- The system shall not overwrite an on-disk change with stale in-memory state.

## Build notes
- Component: `UpdateBanner.tsx` in the renderer; the watch itself detects edits
  to the `I<N>.md` notes and index/JSON files.
- This is the counterpart to reads-hit-the-filesystem (A01/B06) — it's how the
  app finds out an agent or external editor (B07) touched a file.
- Keep it a prompt, not a silent auto-reload, so an in-progress edit isn't
  clobbered without the user's say-so.
