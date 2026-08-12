---
id: H13
epic: H — LLM / agent status via Claude hooks
title: Devin waiting-state (approval/input) via TUI pane-scraping
size: M
requires: [H11, H12, F15]
novel: true
---

## What
Detect when a Devin terminal is **blocked waiting for the human** — a permission
prompt (→ `approval` 🔐, H12's icon) or a question / input request (→ `attention`
⚠️) — by **scraping the Devin TUI pane**, because Devin's lifecycle hooks (H11) do
not expose that state.

A background poller in the main process samples each **live Devin** session's tmux
pane on a short interval, classifies the tail against Devin's waiting-state
signatures, and writes the per-terminal `.llm-status.json` entry (same key and
file H11's hooks use). It is an **overlay** over H11's hook-driven
working/tool/done: it sets `approval`/`attention` while the prompt is on screen
and releases it once the prompt clears, letting the hooks' normal flow resume.
Claude terminals are untouched (Claude's hooks already emit `approval`).

## Why it exists

> H12 gives approval its own 🔐 icon, and H11 was supposed to feed it for Devin
> via a `PermissionRequest` hook. Empirically, in the shipped Devin CLI
> (`3000.3.27`) that hook **never fires**: the interactive "1 Yes … 7 No" prompt
> is not surfaced as any distinct hook event. Verified with marker hooks in
> `.devin/hooks.v1.json` (which *is* loaded — `SessionStart` fires): only
> `PreToolUse`/`PostToolUse`/`Stop` fire, and `PreToolUse`'s stdin is
> byte-identical for an auto-approved `echo` and an approval-gated `rm` (no
> will-prompt flag) — so a hook can neither observe the wait nor tell an
> auto-approved tool from a gated one. The **only** reliable signal is Devin's own
> TUI status line (`Tool approval pending — press q to return`, `Network
> permission pending …`, `Input needed …`, `Devin needs input`) and the inline
> `Approve once` / `always allow … commands` menu. So the state must be read from
> the pane, not from a hook.

Without this, a handed-off Devin can sit silently at a `Yes/No` prompt with the
tab still showing `working` — the user has no at-a-glance cue that Devin is
blocked on them. This is worse under F15/smart mode, where Devin runs unattended
and the human only checks the tabs.

## Acceptance criteria (EARS)
- When a live Devin terminal's pane shows a permission/approval prompt (the
  `Approve once` / `always allow … commands` menu, or a `… approval pending` /
  `… permission pending` status line), the system shall set that terminal's status
  to `approval` (H12's 🔐).
- When a live Devin terminal's pane shows a question / input-needed state
  (`Question pending`, `Input needed`, `Devin needs input`) that is not a
  permission prompt, the system shall set that terminal's status to `attention`.
- When the waiting prompt is no longer present and the poller had set the overlay,
  the system shall release it — restoring `working` only if the overlay it wrote
  is still the current state (so a hook update that already advanced the status is
  never clobbered).
- The poller shall scrape **only** sessions known to be running Devin; Claude
  terminals shall be left entirely to their hooks.
- The poller shall write only on a state change (dedup against the current entry),
  so an on-screen prompt does not rewrite the status file every tick.
- When no Devin session is waiting, the poller shall make no status writes and the
  hook-driven working/tool/done behavior (H11) shall be unchanged.

## Build notes
- **Poller**: a `setInterval` (~1.5 s) in the terminal manager, iterating the live
  session map for `agent === 'devin'`. Reuse the existing `capture-pane -p`
  helper; classify the last ~20 non-blank lines. Cheap no-op when there are no
  Devin sessions; start on app init, clear on quit.
- **Signatures** (case-insensitive, from the shipped TUI):
  - approval: `/Approve once|always allow .*command|Tool approval pending|Network
    permission pending/`
  - attention: `/Question pending|Input needed|Devin needs input/`
  - approval wins if both match.
- **Message**: for approval, lift the pending command from the pane (the
  `└ $ <cmd>` line above the menu) → e.g. `awaiting approval: rm …`; fall back to a
  generic string.
- **Overlay ownership**: keep an in-memory `Map<session, 'approval'|'attention'>`
  of what the poller last applied. On clear, read the current entry; only reset to
  `working` if it still equals the overlay the poller wrote, else just drop the
  tracking (a hook already moved it). Dedup writes by comparing to the current
  entry before writing.
- **Propagation**: write via the shared status-file helper; the existing
  `.llm-status.json` chokidar watcher pushes `llm-status:changed` to the renderer,
  so no new IPC is needed. Writes preserve other sessions' entries (the helper
  re-reads before writing); dedup keeps the write rate at prompt transitions only,
  so races with H11's hook writes stay at the same bound as today's main-side
  writes.
- **Relationship to H11/H12**: this supplements H11 (hooks stay the source for
  working/tool/done) and finally feeds H12's `approval` icon for Devin. If a future
  Devin CLI fires a real `PermissionRequest` hook, that hook can take over and this
  poller can be retired.
