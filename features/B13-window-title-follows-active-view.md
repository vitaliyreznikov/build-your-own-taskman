---
id: B13
epic: B — Kanban board UI
title: OS window title follows active view
size: S
requires: [B12]
novel: false
---

## What
The OS window title reflects the active view — the task (and terminal) currently
in focus — so the title bar names what you're looking at.

## Why it exists
When you run many tasks in parallel, the OS window/title bar is one more place
to route attention — a glance at the title tells you which task is active
without reading the content. It's a small ergonomic that pays off when you're
switching between agents all day.

## Acceptance criteria (EARS)
- When the active view changes to a task, the system shall set the OS window
  title to reflect that task.
- When a terminal within a task becomes the active view, the system shall
  reflect that in the window title.
- Where the active task is resolvable, the system shall include the task's
  human-readable title alongside its id in the window title (not the id alone),
  so the title bar names the work — e.g. `Taskman — I244: Claude session
  history · terminal`.
- When the active task has no title (or it is not resolvable), the system shall
  fall back to the id alone without an empty separator.

## Build notes
- Set in electron-main, driven by the active full-page view (B12) and its
  active terminal.
- Keep the title update reactive to view/terminal focus changes rather than
  computed once at open.
- The task title is looked up from the loaded task snapshot by the active
  terminal's / selected task's id; it stays reactive to title edits.
