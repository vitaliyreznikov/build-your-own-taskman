# MENU — pick your features

Tick what you want, then run [`prompts/00-resolve.md`](prompts/00-resolve.md) to
pull in prerequisites and get an ordered build plan. `size`: S(mall) / M(edium) /
L(arge). ⭐ = one of the two novel ideas. Each row links its feature spec.

> New here? Don't build the menu by hand — grab a ready-made bundle from
> [`PROFILES.md`](PROFILES.md).

---

## Epic A — Markdown data model (AI-first)  · foundation, required by everything

- [ ] **A01** ⭐ Task as a Markdown file (`I<N>.md`) — S — [spec](features/A01-task-as-markdown.md)
- [ ] **A02** Central task index/table (`tasks.md`) — S — [spec](features/A02-tasks-index.md)
- [ ] **A03** Global ordering file (`order.md`) — S — requires A02
- [ ] **A04** Column definitions (`columns.md`: NEXT/IN PROGRESS/BLOCKED/DONE/CLOSED) — S — requires A02
- [ ] **A05** Tags dimension (`boards.md`) — S — requires A02
- [ ] **A06** Atomic file writes (temp-then-rename) — S
- [ ] **A07** Stable ID allocation — S — requires A02
- [ ] **A08** Per-task detail directory (`I<N>/`) — S — requires A01
- [ ] **A09** Git sync engine (debounced background commits) — M — requires A01
- [ ] **A10** External mirror sync (e.g. Obsidian vault) — M — requires A09
- [ ] **A11** Sync status indicator + manual "sync now" — S — requires A09

## Epic B — Kanban board UI  · foundation

- [ ] **B01** Kanban board render (columns of cards) — M — requires A02, A04
- [ ] **B02** Task card (title, priority, project, NextAction, badges) — S — requires B01
- [ ] **B03** Drag-and-drop move between columns — M — requires B01, A03
- [ ] **B04** Quick new-task bar — S — requires A02
- [ ] **B05** Task delete / update fields — S — requires A02
- [ ] **B06** Note detail panel (render/edit the task's Markdown) — M — requires A01
- [ ] **B07** Open note in external editor — S — requires B06
- [ ] **B08** External-link allowlist — S — requires B06
- [ ] **B09** Note search across tasks — S — requires A01
- [ ] **B10** NextAction time model ("back on radar" pointer, sorts rows) — S — requires A02
- [ ] **B11** File watcher + "changed on disk" update banner — M — requires A01
- [ ] **B12** Full-page single-task view (level switcher) — M — requires B01, B06
- [ ] **B13** OS window title follows active view — S — requires B12

## Epic C — Boards / views / filters

- [ ] **C01** Multi-board support (`views.md` + `views/`) — M — requires A02, B01
- [ ] **C02** Main board auto-membership (parent-less tasks show automatically) — S — requires C01
- [ ] **C03** Per-task board (auto-shows its children) — M — requires C01
- [ ] **C04** Add-to-board / remove-from-board controls — M — requires C01, B06
- [ ] **C05** Per-board persisted filters (`filters.json`) — M — requires C01

## Epic D — Task relations

- [ ] **D01** Typed relations table (`relations.md`, e.g. `blocks`, `spawned`) — M — requires A02
- [ ] **D02** "Blocked by" editing in the note panel — M — requires D01, B06
- [ ] **D03** Task typeahead picker for linking — S — requires D01
- [ ] **D04** Extensible relation types — S — requires D01

## Epic E — Native notifications

- [ ] **E01** P0 NextAction desktop notification — M — requires B10
- [ ] **E02** Notification auto-clear (on DONE/CLOSED / priority drop / reschedule) — S — requires E01
- [ ] **E03** Long-delay timer handling (>24-day NextAction) — S — requires E01

## Epic F — In-app terminals  · the pivotal enabler

- [ ] **F01** ⭐ Embedded tmux-backed terminal per task (xterm.js) — L — requires B12
- [ ] **F02** Task-linked session (exports `TASKS_TASK_ID`) — S — requires F01
- [ ] **F03** Content/terminal switcher on a task — S — requires F01
- [ ] **F04** Multiple terminals per task — M — requires F01
- [ ] **F05** Per-terminal ID export (`TASKS_TERM_ID`) — S — requires F04
- [ ] **F06** Terminal tabs — M — requires F04
- [ ] **F07** Drag-and-drop reorder tabs — S — requires F06
- [ ] **F08** High-contrast active tab — S — requires F06
- [ ] **F09** Auto-move task to IN PROGRESS on terminal open — S — requires F01, A04
- [ ] **F10** Single-instance lock + guard closing a tab while an agent runs — M — requires F04
- [ ] **F11** Terminal-state tally in sidebar — S — requires F04, H*

## Epic G — Prompt-parts library

- [ ] **G01** Reusable prompt snippets (`prompts.md`) + browse UI — M — requires A01
- [ ] **G02** Fast terminal insert (⌘K) — M — requires G01, F01

## Epic H — LLM / agent status via Claude hooks  · the supervision cockpit

- [ ] **H01** Live status icons on cards (🤖/⏳/⚠️/✅ + hover message) — M — requires B02
- [ ] **H02** Transient, gitignored status file (`.llm-status.json`) — S — requires H01
- [ ] **H03** Deterministic status via Claude Code lifecycle hooks — L — requires F02, H02
- [ ] **H04** Status markers in answer text (`🤖 working:` … last-match-wins) — M — requires H03
- [ ] **H05** Marker fallbacks (last narration → mechanical tool label) — S — requires H04
- [ ] **H06** Permission-prompt → attention (auto) — S — requires H03
- [ ] **H07** attention/done desktop notifications — S — requires H03, E01
- [ ] **H08** Per-terminal status keying + icon on tab — M — requires F05, H03
- [ ] **H09** Jump to next terminal needing attention — M — requires H08
- [ ] **H10** waiting-vs-attention semantics ("never waiting on me") — S — requires H04

## Epic I — Claude session history & resume

- [ ] **I01** Per-task Claude session history (`.claude-sessions.json`) — M — requires F02, H03
- [ ] **I02** Resume a past session in a new terminal — M — requires I01
