---
id: D03
epic: D — Task relations
title: Task typeahead picker for linking
size: S
requires: [D01]
novel: false
---

## What
When linking one task to another, the user gets an autocomplete field that
suggests matching tasks as they type, rather than requiring them to remember and
type a raw `I<N>` id. It resolves the picked task to its id for the relation.

## Why it exists
Relations reference task ids, but humans think in titles. A typeahead bridges the
two so creating a link (D02) is fast and error-free.

## Acceptance criteria (EARS)
- When the user starts typing in the link field, the system shall show matching
  tasks (by id and/or title) as suggestions.
- When the user selects a suggestion, the system shall use that task's id as the
  relation target.
- The picker shall draw its candidate list from the global task index (A02 via
  D01).

## Build notes
- Component is `TaskTypeahead.tsx`; feed it the task list already available to the
  renderer, matching on id and title.
- It is the input control for D02's add-relation flow; keep it a reusable picker
  so other id-linking spots could use it too.
