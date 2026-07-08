---
id: B06
epic: B — Kanban board UI
title: Note detail panel
size: M
requires: [A01]
novel: false
---

## What
A panel that renders and edits a single task's Markdown note — the body of its
`I<N>.md` file. This is where the free-form content of a task is read and
written from inside the app.

## Why it exists
The note *is* the task — the self-contained Markdown document you point an agent
at. The detail panel is the app's own view onto that document, so you can read
and edit the same text a work session runs against without leaving the board.
It stays a clean human/AI document because the structured fields live elsewhere
(in the index).

## Acceptance criteria (EARS)
- When the user opens a task, the system shall render the Markdown body of its
  `I<N>.md` file in the panel.
- When the user edits the note in the panel and saves, the system shall write
  the new body back to the `I<N>.md` file.
- When the panel reads a note, the system shall read it from disk rather than a
  cached copy, so external edits are reflected.
- When the note is written, the system shall write it atomically.

## Build notes
- Component: `NotePanel`. It reads/writes the `I<N>.md` note body (A01)
  directly.
- The panel edits the note body only — structured fields (B05) and relations
  (epic D) are separate controls that share this panel.
- Reads should hit the filesystem so an agent editing the file elsewhere is
  picked up (pairs with the file watcher, B11).
