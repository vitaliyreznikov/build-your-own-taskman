---
id: B15
epic: B — Kanban board UI
title: In-app terminal follows the colour theme
size: M
requires: [B14, F01]
novel: false
---

## What
The in-app terminal — its xterm screen **and** its chrome (tab strip, command
bar, prompt chips, embedded task-terminal, session history) — follows the B14
colour theme. In Light mode the terminal renders on a light background with dark
text and a light-friendly ANSI palette; in Dark mode it stays the dark scheme it
always had. Switching the theme re-colours every already-open terminal live.

## Why it exists
B14 flipped the CSS-driven surfaces but the terminal stayed dark in every mode,
because the xterm pane's colours are set in JavaScript (a fixed `theme` object)
when a terminal is created — it never listened to `prefers-color-scheme` — and
the surrounding chrome was hard-coded dark. The result was a dark terminal
sitting inside an otherwise light window, which reads as unfinished. Completing
the terminal makes Light mode actually light everywhere the user looks.

## Acceptance criteria (EARS)
- While the effective theme is Light, the system shall render the xterm pane
  with a light background, dark foreground, and an ANSI palette legible on a
  light background.
- While the effective theme is Dark, the system shall render the xterm pane with
  the existing dark scheme (unchanged from before).
- When the theme is switched, the system shall re-apply the matching xterm theme
  to every currently-open terminal without requiring the terminal to be
  reopened.
- In Light mode, the system shall colour the terminal chrome — the tab strip and
  tabs, the command bar and its prompt/input, the clickable prompt chips, the
  embedded task-terminal and its sub-tabs, and the session-history list — to
  match, and the pane's padding gutter shall match the xterm background so no
  colour fringe shows.
- Where the theme is neither explicitly chosen nor overridden, the terminal
  shall follow the OS setting (System mode), consistent with B14.

## Build notes
- The renderer already learns the effective theme for free:
  `window.matchMedia('(prefers-color-scheme: dark)')` reflects
  `nativeTheme.themeSource` (set by B14) and fires `change` when it flips — no
  new IPC.
- xterm side (terminal registry): define a `dark` and a `light` theme (+ their
  background constant), create each terminal with the current scheme's theme,
  and add an apply-scheme pass that sets `term.options.theme` on every live
  registry entry. Register a module-level `matchMedia('… dark)')` change
  listener that runs the apply-scheme pass. Keep the dark theme byte-identical to
  today so Dark mode is unchanged.
- Give the light theme an explicit ANSI palette (dark-on-light red/green/yellow/
  blue/etc.) rather than xterm's defaults, whose bright colours are tuned for a
  dark background and wash out on white. Accept that agent ANSI output is a shade
  less vivid on light than on dark — that's the inherent cost of a light
  terminal, chosen deliberately.
- CSS side: the terminal chrome is dark by *default*; add a
  `@media (prefers-color-scheme: light)` block that re-colours it for Light mode
  (the inverse convention to the rest of the app, which is light by default with
  a dark media block). The pane/`.terminal-view`/`.note-terminal` background must
  equal the light xterm background to avoid fringes.
- Transient overlays that float over the terminal (prompt-search palette, toast,
  over-budget agent chooser) are left on their dark styling — they're momentary
  and read fine over either theme; keeping them out bounds the change.
