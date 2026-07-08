---
id: C05
epic: C — Boards / views / filters
title: Per-board persisted filters (filters.json)
size: M
requires: [C01]
novel: false
---

## What
Each board remembers its own filter state — tags, project, priority — and that
state survives an app restart. Filters are stored per board in a synced
`filters.json` file, read and written through `filters:get` / `filters:set`.

## Why it exists
When you keep several boards, each is used for a different slice of work and
wants a different filter set. Re-applying filters on every launch is friction in
a loop the author runs dozens of times a day, so filter state should be durable
and per-board, not global and ephemeral.

> In the author's words: *"Current filters (tags, project, priority) should be
> saved per board and should survive the app restart"* — and, on where to keep
> it: *"Yes — do it in a synced file."*

## Acceptance criteria (EARS)
- The system shall store a separate filter state (tags, project, priority) for
  each board.
- When the user changes a board's filters, the system shall persist that board's
  filter state to `filters.json`.
- When the app restarts, the system shall restore each board's last-used filters.
- When the user switches boards, the system shall apply the target board's own
  persisted filters, not the previous board's.
- The filter state shall live in a synced file so it travels with the rest of the
  data model.

## Build notes
- Persistence is `filters.json` parsed by `filtersFile.ts`, with `filters:get` /
  `filters:set` IPC. Key the stored state by board slug (C01).
- Unlike the transient status file, `filters.json` is *synced* (committed by the
  git/Obsidian sync engine) — it is user preference that should follow the data,
  per the author's "do it in a synced file."
- Filters select what a board displays; they do not change membership or task
  state. Apply them on top of the board's resolved member set.
