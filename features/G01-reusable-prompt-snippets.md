---
id: G01
epic: G — Prompt-parts library
title: Reusable prompt snippets (prompts.md) + browse UI
size: M
requires: [A01]
novel: false
---

## What
A library of reusable prompt snippets ("prompt parts") stored in a flat
`prompts.md` file, plus a view in the app to browse and manage them. These are
the common instructions the user hands agents over and over — kept once, in one
place, instead of retyped.

## Why it exists
The core loop — spawn an agent, watch it, review it — runs dozens of times a day,
and each spawn carries the same boilerplate instructions (status protocol,
worktree+PR rules, etc.). A durable, AI-legible library of prompt parts removes
that friction, and keeping it as Markdown stays true to the files-as-source-of-truth
model.

> *"Let's find to insert prepared prompt parts to terminal — I need a page to see
> all pieces in use."*

## Acceptance criteria (EARS)
- The system shall store prompt parts in a flat `prompts.md` file (no database).
- The system shall provide a view to browse the prompt parts in use.
- The user shall be able to add, edit, and remove prompt parts, and the system
  shall persist changes to `prompts.md`.
- When `prompts.md` is edited by an external tool, the system shall reflect the
  new content on next read (consistent with the file-as-source-of-truth model).

## Build notes
- Mirror the other data-model files: a small `promptsFile.ts`-style
  parser/serializer over `prompts.md`, read from disk (not a cache) so external
  edits Just Work (A01).
- This is the library and its browse UI only; fast insertion into a terminal is
  G02 (⌘K). Keep the storage format simple enough that the snippet text can be
  piped straight into a terminal.
