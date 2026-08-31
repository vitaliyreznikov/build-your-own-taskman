---
id: B14
epic: B — Kanban board UI
title: Light / dark theme with a manual toggle
size: M
requires: [B01, F01]
novel: false
---

## What
A user-controllable colour theme with three modes — **System** (follow the OS),
**Light**, and **Dark** — chosen from a small segmented control in the sidebar
and remembered across restarts. Picking a mode re-colours the whole app at once
(board, cards, detail panel, notes, the board/PR/prompt/chat/telegram views),
not just the parts that already reacted to the OS setting.

## Why it exists
The app shipped a *partial* dark theme keyed on `@media (prefers-color-scheme:
dark)` — a handful of components (structured-doc view, Telegram, chat rows, task
badge) followed the OS, while the core surfaces (sidebar-adjacent board, cards,
detail/notes panel, prompts and PRs views) stayed light. The result under an
OS-dark setting was a half-dark, inconsistent window, and there was no way to
force a theme independent of the OS. A first-class toggle makes the theme a
deliberate choice and completes the dark surface so the window reads as one
coherent theme in either mode.

## Acceptance criteria (EARS)
- The system shall offer three theme modes — System, Light, Dark — selectable
  from a persistent control in the sidebar.
- When the user selects a mode, the system shall apply it to the entire window
  immediately, without a restart or reload.
- When the app starts, the system shall restore the last-selected mode, and it
  shall do so before first paint so the window does not flash the wrong theme.
- While in System mode, the system shall follow the OS light/dark setting and
  track live changes to it.
- Where a mode is explicitly Light or Dark, the system shall override the OS
  setting (a dark OS with Light selected stays light, and vice-versa).
- In Dark mode, the system shall render every primary surface dark — the board
  columns and cards, the task detail/meta/relations panel, the note editor and
  rendered markdown, the board toolbar, the search-results dropdown, and the
  Prompts and PRs management views — with legible foreground contrast, not only
  the components that already had dark rules.
- The theme preference shall be machine-local (a per-machine display choice),
  and a missing or corrupt preference file shall fall back to System without
  error.

## Build notes
- Drive the theme from electron-main's `nativeTheme.themeSource` (`'system' |
  'light' | 'dark'`). Chromium re-resolves `prefers-color-scheme` in the
  renderer from this value, so **every existing `@media (prefers-color-scheme:
  dark)` rule is reused as-is** — the toggle is a thin control over that single
  source of truth, with no per-element theme class to thread through the tree.
- Persist the choice in a machine-local, gitignored JSON (`.theme.json`,
  alongside the other `.`-prefixed local files); read it at startup and set
  `nativeTheme.themeSource` before creating the window. Set the window
  `backgroundColor` from the effective value (`nativeTheme.shouldUseDarkColors`)
  so the pre-paint frame matches.
- IPC: `theme:get` returns the stored source; `theme:set` validates the source,
  sets `nativeTheme.themeSource`, persists, and returns the effective value.
  Expose both on the preload bridge.
- Renderer: a three-way segmented control in the sidebar reads the current
  source once on mount and calls `theme:set` on change; no CSS class is toggled
  in the renderer — the media query does the work.
- Complete the dark surface by adding the missing `@media (prefers-color-scheme:
  dark)` overrides for the base light surfaces (body, board columns, cards and
  their chips/inputs, the detail panel + note view, the board toolbar, the
  sidebar search dropdown and term-state tally, the Prompts and PRs views).
  Reuse the palette the existing dark blocks already established (near-black
  surfaces `#1c1f24`/`#1e1f22`, panel `#23272e`, borders `#3a3f47`, text
  `#ddd`/`#a8b3bf`) so the new rules match the ones that shipped earlier.
- The sidebar and in-app terminals are already dark in both modes; leave them.
