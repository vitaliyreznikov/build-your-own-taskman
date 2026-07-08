---
id: A02
epic: A — Markdown data model
title: Central task index/table (tasks.md)
size: S
requires: []
novel: false
---

## What
A single Markdown file, `tasks.md`, holds one row per task in a table. Each row
carries the task's *structured* fields: `id`, `title`, `column`, `priority`,
`project`, `parent`, `board`, and `NextAction`. This is the index over the flat
pile of `I<N>.md` notes — the place the board reads to know what exists and where
each task sits, without opening every note.

## Why it exists
The note (A01) is a clean, schema-free human/AI document; the *machine* fields
have to live somewhere that's still plain and agent-legible. `tasks.md` is that
somewhere. Keeping state in a flat Markdown table — not a database, not YAML in
each note — means an agent (or the human, or a diff) can read and edit the whole
project's structure at a glance, which is the entire point of the AI-first data
model.

> The two ideas the author considers genuinely fresh start here: *"work with
> tasks as MD files so they are AI first."* The index is what keeps state as flat
> files instead of a DB.

## Acceptance criteria (EARS)
- When a task is created, the system shall append a row to `tasks.md` with its
  `id` and `title` and defaults for the remaining fields.
- When a task's column, priority, project, parent, board, or NextAction changes,
  the system shall update that task's row in `tasks.md`.
- When the board renders, the system shall derive the set of tasks and their
  placement from `tasks.md` (in combination with A03/A04), not from the notes.
- When a task is deleted, the system shall remove its row from `tasks.md`.
- The system shall keep the note body (A01) out of `tasks.md`; the table holds
  only structured fields.

## Build notes
- One row = one task; `id` is the join key back to `I<N>.md` (A01) and to
  `order.md` (A03), `columns.md` (A04), `boards.md` (A05).
- Column values are constrained by `columns.md` (A04); `board`/tags by `boards.md`
  (A05); `NextAction` is the `YYYY-MM-DD HH:MM` "back on radar" pointer.
- `parent` is a task id (or empty) — it drives main-board auto-membership and
  per-task child boards in the later boards/views epic.
- Always write through the atomic writer (A06) so a crash mid-write can't corrupt
  the whole project index.
- Deliberately keep it a *table*, not per-note frontmatter: one file to scan, one
  file to diff, trivial for an agent to rewrite.
