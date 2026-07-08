---
id: B04
epic: B — Kanban board UI
title: Quick new-task bar
size: S
requires: [A02]
novel: false
---

## What
An inline bar for creating a task without leaving the board. Type a title, hit
enter, and a new task is created — a fresh `I<N>.md` note plus a row in
`tasks.md`.

## Why it exists
Capturing a task has to be near-zero friction, because it happens dozens of
times a day in the middle of other work. An always-available inline bar means
a thought becomes a task in one gesture, keeping the board a faithful picture
of everything in flight.

## Acceptance criteria (EARS)
- When the user types a title in the new-task bar and confirms, the system
  shall create a new task with that title (via `tasks:create`).
- When a task is created, the system shall allocate the next `I<N>` id, write
  its `I<N>.md` note, and add its row to `tasks.md`.
- When a task is created, the system shall show it on the board immediately.
- When the title is empty, the system shall not create a task.

## Build notes
- Component: `NewTaskBar`. It calls `tasks:create` in electron-main.
- Id allocation is handled by `ids.ts` (A07) — the bar should not invent ids
  itself.
- Creation writes both the note file (A01) and the index row (A02); default
  column/priority come from the data model, not the bar.
