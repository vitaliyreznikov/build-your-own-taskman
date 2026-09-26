---
id: A14
epic: A — Markdown data model
title: Category dimension (rename boards.md tags → categories.md)
size: M
requires: [A05, A13]
novel: false
---

## What
The per-task "area of life/work" dimension introduced by A05 is renamed from
**board/tag** to **category** everywhere: the vocabulary file becomes
`categories.md`, the `tasks.md` column becomes `Category`, the task field becomes
`category`, and the UI says "Category" / "Categories" (card field, sidebar filter,
new-task picker, terminal-strip filter "— no category —"). A task carries
**exactly one** category.

## Why it exists
A05 shipped with loose historic naming: the file was `boards.md`, the column
`Board`, the filter "Tags". Both words mislead:
- **Board** already means a Kanban *surface* (`views.md`, `views/`) — one word for
  two unrelated things made every conversation and every agent prompt ambiguous.
- **Tag** implies several per task; the field was always single-valued.

"Category" says what it is: one bucket per task, from a small fixed vocabulary.

## Acceptance criteria (EARS)
- The system shall read the category vocabulary from `categories.md`.
- If `categories.md` does not exist and `boards.md` does, the system shall read
  the vocabulary from `boards.md`, and on its next vocabulary write shall write
  `categories.md` and remove `boards.md`.
- The system shall read each task's category from the `Category` column of
  `tasks.md`; if the table has a `Board` column and no `Category` column, the
  system shall read that column as the category.
- When the system writes `tasks.md`, it shall write the header `Category`
  (never `Board`).
- A task shall carry exactly one category; the UI shall present it as a single
  choice, not a multi-select.
- The UI shall label the dimension "Category" (singular, on a task) and
  "Categories" (the sidebar filter); the word "board" shall refer only to board
  surfaces (views).
- When persisted per-view filters or the terminal-strip hidden set were saved
  under the old key, the system shall read them as the category filter, so a
  rename does not reset what the user had hidden.
- When a task is created with no category, the system shall use the A13
  default (`personal`).

## Build notes
- Back-compat is read-side only: accept the old file/column/keys, always write the
  new ones. After one write cycle the old names are gone from the data. This lets
  a git-synced second machine running an older build keep working until it
  updates (it will see an unknown `Category` column — so update all machines
  together, or keep old builds read-only).
- Renaming a category *value* (e.g. one `job` becoming a more specific name) is a
  plain data edit: change the line in `categories.md`, rewrite the matching cells
  in `tasks.md`, and rename it in any persisted filter sets — otherwise a filter
  that had the old value checked silently hides the renamed tasks. Do it with the
  app stopped.
- External readers of `tasks.md` (time trackers, scripts) that parse by column
  **position** survive the header rename; ones that key on a value need the new
  value added.
- Internally rename the field (`task.board` → `task.category`) too — leaving the
  old identifier keeps the two meanings of "board" tangled in code, which is the
  thing this feature removes.
