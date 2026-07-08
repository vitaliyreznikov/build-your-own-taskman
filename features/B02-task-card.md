---
id: B02
epic: B — Kanban board UI
title: Task card
size: S
requires: [B01]
novel: false
---

## What
The card is the unit shown in each board column. It renders a task's title,
priority, project, and NextAction, plus a row of badges (later the live LLM
status icon lives here). Clicking it opens the task; it is the thing you drag
between columns.

## Why it exists
The card is the smallest glanceable summary of a task — enough to decide
whether it needs you without opening it. In a workflow built around
supervising many tasks in parallel, the card is where the human's eye lands
first, so it has to carry exactly the fields that drive that decision and no
more.

## Acceptance criteria (EARS)
- When a task is rendered on the board, the system shall show its title,
  priority, project, and NextAction on the card.
- The system shall render a badge row on the card for status/metadata markers.
- When the user clicks a card, the system shall open that task (note panel /
  full-page view).
- When a shown field is empty in `tasks.md`, the system shall omit it cleanly
  rather than rendering a blank slot.

## Build notes
- Component: `Card.tsx`. Fields are read straight from the task's row in
  `tasks.md` (A02) — the card renders index data, not the Markdown note body.
- The badge row is the anchor point for the live LLM status icon (epic H); keep
  it as a slot so that icon can drop in later without reworking the card.
- Keep the card a presentation component: dragging (B03) and field edits (B05)
  are wired at the board/panel level, not inside the card.
