---
id: B20
epic: B — Kanban board UI
title: Compact task panel — essentials on top, the rest below the content
size: S
requires: [B06, B05, O06]
novel: false
---

## What
The Content tab of a task's panel is **one scrolling column**, laid out by how
often each control is used:

1. **Top, compact:** the title; one row with Category · Priority · Column ·
   Project; one row with Start · Due, each with its quick-set buttons on the
   same line; the per-terminal agent status banner ("Attention needed …").
2. **The task's content:** the v2 subgoal tree or the markdown note.
3. **Below the content:** Repeat, Follow-up, Detail (note / v2, convert),
   Autorun, the manual LLM-status editor (collapsed by default), Boards,
   Children, Blocked by, created/updated stamps, Delete.

The rows wrap in a narrow side panel and line up in a full-page view.

## Why it exists
The metadata block sat *above* the scrolling content and grew with every
feature (Due, Repeat, follow-up, autorun, LLM status, boards, relations), so the
subgoals got only the leftover strip at the bottom of the screen.

> *"съэкономь место здесь — куча малоиспользуемых функций едят много места;
> что-то сожми, что-то перенеси ниже списка подзадач"*

## Acceptance criteria (EARS)
- The system shall render the Content tab (view mode) as a single scroll
  container holding the top block, the content and the lower block, in that
  order.
- The system shall show Category, Priority, Column and Project in one row that
  wraps when the panel is too narrow.
- The system shall show Start and Due side by side, with their quick-set buttons
  inline after each input.
- The system shall show Repeat, Follow-up, Detail, Autorun, LLM status, Boards,
  Children, Blocked by, the stamps and Delete after the content.
- The system shall collapse the manual LLM-status editor by default; the
  per-terminal status banner stays at the top.
- While the markdown note is in edit mode, the system shall show the top block
  and the editor only.
- No control shall be removed and no field's behaviour shall change.

## Build notes
- Render one meta component in two parts (`top` / `bottom`) so its draft state
  and handlers stay in one place.
- Inside the scroll container the content stops being its own `flex: 1;
  overflow: auto` box — otherwise the lower block is pinned to the bottom again.
