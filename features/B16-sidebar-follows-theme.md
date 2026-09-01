---
id: B16
epic: B — Kanban board UI
title: Sidebar follows the colour theme
size: S
requires: [B14]
novel: false
---

## What
The left sidebar follows the B14 colour theme: light in Light mode, dark in Dark
mode — instead of being a permanently-dark rail. Covers the whole sidebar
content: section headings, board-nav items (hover/active), the search box, the
Claude-usage and sync-status widgets, and the theme toggle itself.

## Why it exists
B14/B15 themed the board, panels and terminal, but the sidebar was hard-coded
dark in both modes (a deliberate "dark rail" like Slack/VS Code). In Light mode
that left a dark rail against an otherwise light window, which read as a missed
spot rather than a design choice. Making the sidebar follow the theme gives one
coherent surface.

## Acceptance criteria (EARS)
- While the effective theme is Light, the system shall render the sidebar and all
  its content on a light surface with legible foreground contrast.
- While the effective theme is Dark, the system shall render the sidebar with the
  existing dark scheme (unchanged).
- When the sidebar is light, the system shall keep it visually separated from the
  adjacent board area (a subtle edge), so the two light surfaces don't merge.
- The theme-toggle control shall remain legible and correctly indicate the
  active mode in both themes.

## Build notes
- Same convention as the rest of the app: the sidebar is dark **only** where its
  rules are the default; recolour it for Light mode with a
  `@media (prefers-color-scheme: light)` block (the terminal chrome B15 added
  uses the same inverse-of-default pattern). Dark mode stays byte-identical.
- Add a light right-edge separator to the sidebar (border) so the light rail is
  distinct from the light board.
- The term-state tally and the search-results dropdown are already light by
  default (they had dark overrides added in B14), so they need no new light
  rules — only the genuinely dark sidebar chrome does.
- Keep the semantic usage-delta colours (headroom/ahead/full) — they read on
  both surfaces.
