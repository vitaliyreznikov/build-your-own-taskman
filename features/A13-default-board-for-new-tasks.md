---
id: A13
epic: A — Markdown data model
title: Default board for new tasks (personal)
size: S
requires: [A05, A07]
novel: false
---

## What
When a task is created **without an explicit board**, the system assigns it a
single, fixed default board — `personal` — instead of "whatever tag happens to
be first in `boards.md`". This one default is shared by every no-board creation
path: the `+ Add` bar's neutral fallback, the `create-task` control-API verb
(P03), and any other `createTaskInternal` caller. An explicitly-supplied board
is always honored unchanged.

## Why it exists
The board a new task lands on was an accident of list order: creation defaulted
to `boards[0]`, the first entry in `boards.md` (historically `job`). So every
ad-hoc task filed with no board — including ones an agent files via
`taskman create-task` — kept landing on the *work* board and had to be
re-tagged by hand. Most tasks the human files ad-hoc are life/family
(`personal`) tasks, not work. Making the creation default an explicit `personal`
decouples "which tag is listed / shown first" from "where an untagged new task
lands", so re-ordering the board vocabulary never silently moves the default.

## Acceptance criteria (EARS)
- When a task is created and no board is specified, the system shall set its
  board to `personal`.
- When a task is created with an explicit board, the system shall use that board
  unchanged.
- When exactly one board is active in the filter, the `+ Add` bar shall continue
  to create on that filtered board; otherwise it shall default to `personal`
  (no longer to the first configured board).
- When the order of `boards.md` changes, the new-task default shall not change.
- Chat-created tasks (already defaulting to `personal`) shall be unaffected and
  now consistent with the shared default.

## Build notes
- `personal` must exist in `boards.md` as a valid tag (A05) — it does. The
  default is a plain string, never `boards[0]`.
- Single source of truth: one `DEFAULT_BOARD = 'personal'` constant so the main
  process (`createTaskInternal`) and the renderer (`NewTaskBar`) agree. The old
  `?? 'job'` empty-list fallbacks become `?? DEFAULT_BOARD`.
- No storage/schema change: the board still rides the `board` field in
  `tasks.md` (A05). No migration — existing tasks keep their board.
- The chat-create path already hardcodes `personal`; leave it, it now matches
  the shared default.
