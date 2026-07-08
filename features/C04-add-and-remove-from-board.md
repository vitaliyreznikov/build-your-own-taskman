---
id: C04
epic: C — Boards / views / filters
title: Add-to-board and remove-from-board controls
size: M
requires: [C01, B06]
novel: false
---

## What
From a task's detail panel, the user can add the task to any board and see the
list of boards the task is currently on, each with a control to remove it. Adding
to a not-yet-existing board creates the board. This is the manual side of the
additive-membership model.

## Why it exists
Auto-membership (C02, C03) covers the common cases, but the author still wanted
explicit, per-task control from where he works — the detail panel — to promote a
child onto the main board or drop a task from a board he curates.

> In the author's words: *"They should not be created at advance but I need
> button in the task tab."*

## Acceptance criteria (EARS)
- When the user invokes add-to-board from a task's detail panel, the system shall
  add the task's id to the chosen board's membership file.
- When the chosen board does not yet exist, the system shall create it (new
  `views/<slug>.md`) and then add the task.
- The detail panel shall list every board the task is currently a member of.
- When the user removes the task from a board, the system shall delete its id
  from that board's membership file.
- Removing a task from a board shall not change the task's global state or its
  membership on other boards.

## Build notes
- IPC surface: `views:addMember` / `views:create` for adding, `views:removeMember`
  / `views:delete` for removal; the panel reads the task's board memberships to
  render the overview.
- Only manual pins are removable — a child shown on its parent's board by the
  auto-rule (C03) is not "a member" to remove; removal edits the membership file,
  not the auto-rule.
- Lives in the NotePanel/detail UI (B06); keep writes atomic so concurrent board
  edits don't corrupt a membership file.
