---
id: A07
epic: A — Markdown data model
title: Stable ID allocation
size: S
requires: [A02]
novel: false
---

## What
A small allocator (`ids.ts`) hands out the next task id as `I<N>` with a
monotonically increasing integer `N`. The id, once assigned, is stable for the
life of the task and is the filename of its note (`I<N>.md`) and the join key
across every data file.

## Why it exists
Everything in the model is joined by id: the note, the index row, the order
entry, boards, relations, per-task directories, terminal session names
(`task-I<N>`), status keys. That only works if ids are stable and never reused, so
a reference to `I246` always means the same task even after deletions. A trivial,
monotonic allocator is the cheapest way to guarantee that.

## Acceptance criteria (EARS)
- When a task is created, the system shall allocate the next unused integer `N`
  and assign the id `I<N>`.
- When it allocates `N`, the system shall derive the maximum from *both* the index
  rows (`tasks.md`, A02) *and* the note files present on disk (`I<n>.md`, A01), so
  an id an external agent has already claimed by writing a note is never reused —
  even before that agent's index row has landed.
- Allocated ids shall be monotonically increasing and shall not be reused, even
  after a task is deleted.
- Once assigned, a task's id shall never change.
- The allocated id shall be usable directly as the note filename (`I<N>.md`, A01)
  and as the join key in `tasks.md` (A02) and all other data files.

## Build notes
- Keep the id → filename mapping trivial: id is `I` + integer, filename is
  `<id>.md` (mirrors A01).
- Derive "next N" from the existing set of ids (e.g. max + 1) so allocation stays
  correct even if the app restarts or files were edited externally — don't rely on
  an in-memory counter alone.
- The "existing set of ids" must union `tasks.md` rows *and* the `I<n>.md` note
  files on disk. A task created externally (by an agent) is a note write plus an
  index-row write that are not atomic together; if allocation looks at `tasks.md`
  alone it can hand out an id whose note already exists, clobbering that task.
  Scan the data directory for `I<n>.md` and take the max across both sources.
- Do not recycle ids of deleted tasks; downstream references (relations, session
  history, terminal session names) assume ids are permanent.
