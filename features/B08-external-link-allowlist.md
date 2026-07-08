---
id: B08
epic: B — Kanban board UI
title: External-link allowlist
size: S
requires: [B06]
novel: false
---

## What
Opening links found in a task's note in the system browser, gated by a
scheme allowlist (`notes:openExternal`) so only vetted URL schemes are handed
to the OS.

## Why it exists
Notes are free-form Markdown that can contain arbitrary links, and an agent may
write them. Handing an arbitrary URL to the OS is a security surface, so
external opens go through an allowlist — the app decides which schemes are safe
to launch rather than trusting whatever the note contains.

## Acceptance criteria (EARS)
- When the user clicks a link in a note, the system shall open it externally
  only if its scheme is on the allowlist (via `notes:openExternal`).
- When a link's scheme is not on the allowlist, the system shall not open it.

## Build notes
- Wiring: `notes:openExternal` with the scheme allowlist enforced in `main.ts`.
- Keep the allowlist decision in electron-main, not the renderer, so the
  renderer can never open an arbitrary scheme directly.
- Applies to links surfaced from the note panel (B06).
