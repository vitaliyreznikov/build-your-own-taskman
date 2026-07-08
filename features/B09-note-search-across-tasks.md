---
id: B09
epic: B — Kanban board UI
title: Note search across tasks
size: S
requires: [A01]
novel: false
---

## What
Full-text search across all task notes (`notes:search`), so you can find a task
by something written in its body, not just its title.

## Why it exists
Context lives in files, not in a chat that evaporates — each task note is a
durable record. Search across those notes is how you get that context back:
point at what you remember writing and jump to the task, even months later.

## Acceptance criteria (EARS)
- When the user searches, the system shall match the query against the body
  text of the `I<N>.md` notes (via `notes:search`).
- When there are matches, the system shall return the tasks whose notes match
  so the user can open them.

## Build notes
- Wiring: `notes:search` IPC handler in electron-main scans the `I<N>.md` notes
  (A01).
- Searches note bodies on disk — the notes are the corpus, consistent with the
  file-as-source-of-truth model.
