---
id: A08
epic: A — Markdown data model
title: Per-task detail directory (I<N>/)
size: S
requires: [A01]
novel: false
---

## What
Beside a task's note (`I<N>.md`) a task may optionally have a directory named
`I<N>/` holding overflow material for that task: findings, scripts, data dumps,
patches, review notes — anything too big or too numerous to inline in the note.
The note links into this directory with relative links.

## Why it exists
A task should be a durable, self-contained workspace an agent can be pointed at
cold — context lives in files, not in a chat that evaporates. Keeping the note
clean while still capturing everything an agent produced means giving each task a
place to put its artifacts, right next to the note, addressable by relative path.

> In the author's words: *"Just say that every task can create directory
> .../I<id>/... to keep their details."*

## Acceptance criteria (EARS)
- A task shall be able to have an optional directory `I<N>/` alongside its note
  `I<N>.md`.
- When the note links to material in its directory, the system shall use relative
  links so the pair stays portable and moves/mirrors together.
- The directory shall be optional — a task with no overflow material has no
  directory, and that is a normal state.
- Content placed in `I<N>/` shall be treated as part of the task's data and
  included in sync (A09).

## Build notes
- It's a convention, not a schema: no required layout inside `I<N>/`. Agents drop
  whatever they need (e.g. `pr-<n>.md`, `review-<n>.md`, `*.patch`).
- Use relative links from the note so an external mirror (A10) or a git checkout
  resolves them the same way the app does.
- The convention predates the docs that formalized it; keep it lightweight —
  creating the directory on demand, not up front for every task.
