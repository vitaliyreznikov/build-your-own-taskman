---
id: B17
epic: B — Kanban board UI
title: Board picker modal on new-task "+"
size: S
requires: [B04, A05, A13]
novel: false
---

## What
When the user adds a task from the quick new-task bar (`+ Add` / Enter), the
system shows a small modal asking **which board** the task should go on
(`job` / `personal` / `brootto` / …) before the task is created. The modal
lists every configured board, pre-selects the board the bar would otherwise
have defaulted to, and creates the task on the board the user picks. Cancelling
(Esc / backdrop / Cancel) creates nothing and keeps the typed title so the user
can retry.

## Why it exists
The new-task bar creates instantly onto a single default board (A13:
`personal`, or the sole active filter). That is right for rapid capture, but
most tasks then have to be re-tagged by hand because the human knows at capture
time whether the thought is work (`job`), family business (`personal`),
`brootto`, job-search, learning, etc. A one-gesture board choice at creation —
defaulted, so Enter-Enter is still fast — removes the re-tagging step and keeps
the board vocabulary honest without re-ordering `boards.md`.

## Acceptance criteria (EARS)
- When the user confirms the new-task bar with a non-empty title, the system
  shall open a board-picker modal instead of creating the task immediately.
- The modal shall list every configured board (from `boards.md` / `view.boards`)
  as a selectable option.
- When the modal opens, the system shall pre-select the board the bar would
  otherwise default to: the parent's board when adding on a task's board, else
  the sole active board filter, else the A13 default (`personal`).
- When the user picks a board and confirms, the system shall create the task on
  that board (all other behaviour — child linkage on a task board, view-member
  add on a user board, document type — unchanged).
- When the user cancels (Esc, backdrop click, or Cancel), the system shall not
  create a task and shall preserve the typed title and selected document type.
- When the title is empty, the system shall neither open the modal nor create a
  task.

## Build notes
- Renderer-only: the modal lives in `NewTaskBar`. The board list comes from
  `view.boards` (same source the Sidebar uses); no new IPC, no main change.
- The default-board computation (A13: single-filter → `personal`; parent board
  on a task board) is unchanged — it now seeds the modal's pre-selection instead
  of being passed straight to `createTask`.
- Keyboard: Enter in the title field opens the modal; in the modal, Enter
  confirms the highlighted board and Esc cancels. The modal is a transient
  overlay (styled dark, like the agent chooser) — see B14's note that transient
  overlays stay dark by design.
- No storage/schema change: the board still rides the `board` field in
  `tasks.md` (A05).
