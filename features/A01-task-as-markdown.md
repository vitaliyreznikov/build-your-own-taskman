---
id: A01
epic: A — Markdown data model
title: Task as a Markdown file
size: S
requires: []
novel: true
---

## What
Every task is a standalone Markdown file named `I<N>.md` (e.g. `I246.md`), living
in one flat directory. The file's body is free-form Markdown notes about the task.
There is no database — this file is the canonical record of the task's content.

## Why it exists
The task file is meant to be the *interface to an agent*. Every work session
starts literally by pointing an LLM at the file: "work on `I246.md`". For that to
work, the task has to be a plain, self-contained, machine-legible document — not a
row hidden in a database. This is the foundational decision that makes the whole
app "AI-first": the file **is** the prompt.

> Design intent, in the author's words: *"work with tasks as MD files so they are
> AI first"* — and every session begins "Please work on task /…/I<id>.md".

## Acceptance criteria (EARS)
- When the user creates a task, the system shall write a new `I<N>.md` file with a
  unique, monotonically increasing `N`.
- The file shall contain the task's notes as plain Markdown, with no required
  schema beyond a title heading.
- When the user opens a task, the system shall read its content directly from the
  `I<N>.md` file on disk.
- When the file is edited by an external tool (editor, another agent), the system
  shall reflect the new content on next read (no cached/DB copy wins over the file).
- The system shall not store task body content anywhere other than the `I<N>.md`
  file.

## Build notes
- Keep the filename → id mapping trivial: id is `I` + integer, filename is
  `<id>.md`. See A07 for allocation of the next id.
- Body is intentionally schema-free. Structured fields (priority, column, etc.)
  live in the index (A02), *not* in the note — so the note stays a clean human/AI
  document.
- Pair with A08 (`I<N>/` directory) for tasks that need overflow material
  (findings, scripts, data dumps) linked from the note with relative links.
- Reads should hit the filesystem, not a cache, so external edits Just Work — this
  is what lets an agent edit a task file and have the app pick it up.
