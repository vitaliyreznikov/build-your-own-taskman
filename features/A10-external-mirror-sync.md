---
id: A10
epic: A — Markdown data model
title: External mirror sync (Obsidian vault)
size: M
requires: [A09]
novel: false
---

## What
Alongside git commits, the sync engine mirrors the task data into an external
Markdown vault (e.g. `~/workspace/obsidian`), so the same notes and per-task
directories are browsable and editable in an outside Markdown tool. The mirror
runs on the same debounced/periodic cadence as the git sync.

## Why it exists
Because tasks are plain Markdown, they don't have to live only inside the app —
they can also be first-class documents in the human's existing note system. The
mirror is a direct payoff of the AI-first, files-as-truth design: the same files
serve the app, an agent, git history, and a personal knowledge base at once.

## Acceptance criteria (EARS)
- When the sync engine runs, the system shall mirror the task Markdown (notes and
  per-task `I<N>/` directories) into the configured external vault directory.
- The mirror shall run on the same debounced/periodic cadence as the git sync
  (A09), not as a separate manual step.
- Relative links inside notes shall resolve correctly in the mirrored location
  (A08), so the mirror is browsable standalone.
- When mirroring fails, the failure shall be surfaceable to the status indicator
  (A11) without stopping git sync.

## Build notes
- Reuse the same trigger as A09 (debounce + periodic) rather than a second
  independent scheduler; the mirror is a second output of one sync pass.
- Because A08 uses relative links, the note/`I<N>/` pair copies cleanly into the
  vault — preserve directory structure so links stay valid.
- Make the vault path configurable; the reference implementation targets an
  Obsidian vault under `~/workspace/obsidian`.
- Keep transient/gitignored files (status JSON) out of the mirror, same as git.
