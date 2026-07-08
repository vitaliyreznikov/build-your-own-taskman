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
- Do not recycle ids of deleted tasks; downstream references (relations, session
  history, terminal session names) assume ids are permanent.
