---
id: C02
epic: C — Boards / views / filters
title: Main board auto-membership
size: S
requires: [C01]
novel: false
---

## What
The main board populates itself: every parent-less task appears on it
automatically, with no membership entry required. Child tasks do *not* appear by
default, but can be pinned onto the main board explicitly via `views/main.md`.

## Why it exists
The default board should show "the things I'm actually tracking" without the
author curating a list by hand. Top-level tasks are those things; subtasks are
implementation detail that belongs on their parent's board, not cluttering the
main one — unless the author deliberately promotes one.

> In the author's words: *"All tasks go to main board by default but children
> tasks."*

## Acceptance criteria (EARS)
- When a task has no parent, the system shall show it on the main board without
  any entry in `views/main.md`.
- When a task has a parent, the system shall not show it on the main board by
  default.
- When a child task id is listed in `views/main.md`, the system shall show that
  child on the main board (a manual pin).
- The system shall keep task state global — pinning a child to the main board
  shall not change its column, priority, or membership on other boards.

## Build notes
- Membership for the main board is a union: (all tasks where `parent` is empty in
  `tasks.md`) ∪ (ids explicitly listed in `views/main.md`).
- `views/main.md` uses the same membership-file format as any other board (C01);
  it just carries the auto-rule for parent-less tasks on top.
- The parent field is read from the A02 index, not the note body, so this rule is
  a pure function of the index plus one small file.
