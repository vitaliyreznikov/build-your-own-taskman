---
id: B07
epic: B — Kanban board UI
title: Open note in external editor
size: S
requires: [B06]
novel: false
---

## What
A command that opens the task's `I<N>.md` file in the user's external editor,
straight from the detail panel (`notes:openInEditor`).

## Why it exists
Because a task is just a Markdown file on disk, it should open in whatever
editor you already use — the app doesn't have to be the only way to work on the
note. This is the file-as-source-of-truth model paying off: the same file the
app shows is a real file any tool can edit.

## Acceptance criteria (EARS)
- When the user invokes "open in editor" on a task, the system shall open that
  task's `I<N>.md` file in the configured external editor
  (via `notes:openInEditor`).
- When the file is edited externally and saved, the system shall reflect the
  new content on next read.

## Build notes
- Wiring: `notes:openInEditor` IPC handler in electron-main launches the editor
  on the task's `I<N>.md` path.
- Works because the note is a real file (A01/B06); no export/import step.
- Pairs with the file watcher (B11) so edits made in the external editor surface
  in the app.
