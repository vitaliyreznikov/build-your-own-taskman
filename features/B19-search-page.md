---
id: B19
epic: B — Kanban board UI
title: Search as its own page
size: S
requires: [B09, B12]
novel: false
---

## What
Search results get a **dedicated page** in the main area — a sibling of Board /
Terminal / PRs — instead of a dropdown under the sidebar input. The sidebar keeps
only a compact search box: typing in it switches the main area to the Search page,
which shows the full result list (id, title, status, where it matched, the whole
snippet) with room to read it. Clicking a result opens that task no matter which
view was active before, including the Terminal view.

## Why it exists
The sidebar dropdown (B09's first UI) was too narrow to read snippets, and it grew
over the rest of the sidebar — terminal states, the board filter, Claude limits —
hiding functions the user still needed while searching. Worse, clicking a hit only
set the selected task: when the main area was showing the Terminal view the click
visibly did nothing, because the task panel lives in the board view. A page of its
own fixes all three: it has the width for snippets, it covers nothing, and "open
this result" is an explicit navigation to the task.

## Acceptance criteria (EARS)
- The sidebar search box shall not render a results list of its own; the rest of
  the sidebar shall stay visible and usable while a query is typed.
- When the user types a non-empty query in the sidebar search box, the system shall
  switch the main area to the Search page showing that query's results.
- The Search page shall have its own search input bound to the same query, and a
  "Search" entry in the sidebar view navigation.
- Each result shall show the task id, title, current column, where it matched
  (id / title / note / doc) and the full snippet, unclipped.
- When the user clicks a result (or presses Enter with a result highlighted), the
  system shall open that task — switch to the board view and select it (full-page
  when B12's full-page mode is on) — regardless of which view was active.
- While the Search page is focused, the system shall let Up/Down move the
  highlighted result and Esc clear the query.
- The Search page shall show more results than the dropdown did (at least 100).

## Build notes
- Renderer: add `'search'` to the active-view union; keep the query in the store so
  the sidebar box and the page share it. Opening a result = `setSelected(id)` +
  `setActiveView('board')`.
- The `notes:search` IPC (B09) is unchanged apart from its result cap.
