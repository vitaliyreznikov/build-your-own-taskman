---
id: B18
epic: B — Kanban board UI
title: Board picker on the terminal-strip "+"
size: S
requires: [B17, F13, F20]
novel: false
---

## What
The Terminal view's **"+"** button (F13: reserve a task id + spawn an agent that
interviews the user and fills in the task) now opens the **same board-picker
modal** as the board-view new-task bar (B17) before it creates the placeholder
task. The user chooses which board (`job` / `personal` / `brootto` / …) the new
task+terminal lands on, instead of it silently defaulting to `personal` (A13).
Cancelling (Esc / backdrop / Cancel) creates nothing — no placeholder task and no
terminal.

## Why it exists
Two features shipped independently and collided. F13's "+" always created on the
A13 default board (`personal`); F20 then let the user *hide* a board's terminal
tabs from the strip. Together they produced a silent failure: clicking "+" while
`personal` terminals were hidden created the task+terminal on `personal` — which
was immediately filtered out of the strip — so the button appeared to do
**nothing**. Routing the "+" through B17's picker makes the destination an
explicit, one-gesture choice (like the board-view bar already is), and — by
pre-selecting a board whose terminals are currently *visible* — guarantees the
new tab shows up when the user just confirms the default. It also gives the
terminal "+" the same board-choice ergonomics the board-view "+ Add" got in B17.

## Acceptance criteria (EARS)
- When the user clicks the Terminal-strip "+", the system shall open a
  board-picker modal instead of creating the task and terminal immediately.
- The modal shall list every configured board (from `boards.md` / `view.boards`)
  as a selectable option.
- When the modal opens, the system shall pre-select a sensible default board:
  the A13 default (`personal`) when its terminals are currently visible under the
  F20 filter, else the first board whose terminals are visible, else the first
  configured board.
- When the user picks a board and confirms, the system shall create the
  placeholder task **on that board** and then proceed exactly as F13 does today
  (route through the F15/F17 agent chooser when over budget, create the task in
  `IN PROGRESS`, open the linked terminal, auto-start the chosen agent).
- When the user cancels (Esc, backdrop click, or Cancel), the system shall create
  neither a placeholder task nor a terminal.
- The board-view new-task bar (B17) shall continue to use the same picker with no
  behaviour change (the component is shared, not duplicated).

## Build notes
- Renderer-only. Extract B17's inline `BoardPicker` overlay from `NewTaskBar` into
  a shared `components/BoardPicker.tsx` and import it in both `NewTaskBar` and
  `TerminalView`; markup/CSS (`board-picker*`, dark transient overlay per B14) is
  unchanged so B17 is byte-for-byte the same modal.
- In `TerminalView`, `newTerminal()` gains a `board: string` parameter that it
  passes straight into `window.tasksApi.createTask({ title:'New task…',
  column:'IN PROGRESS', board })`. The "+" `onClick` no longer calls
  `newTerminal` directly — it opens the picker (`pickBoard` state), and the
  picker's `onPick` calls `newTerminal(board)`.
- The picker still opens *before* the F15/F17 agent chooser, so an over-budget
  user sees board → agent → launch; a healthy user sees board → launch. The
  placeholder task is still created inside the `maybeChooseAgent` callback so
  cancelling the *agent* chooser also leaves nothing behind (unchanged from F13).
- Default pre-selection reuses the F20 `hiddenTermBoards` set already in the store
  to prefer a visible board — no new IPC, no storage/schema change. The board
  rides the existing `board` field in `tasks.md` (A05).
- Deliberately scoped to A only: this does not auto-unhide a board (that was the
  rejected "B" option). Picking a *hidden* board on purpose still lands there and
  stays filtered — but the visible-default pre-selection means confirming the
  default always shows the new tab.
