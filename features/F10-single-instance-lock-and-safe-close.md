---
id: F10
epic: F — In-app terminals
title: Single-instance lock and guard closing a tab while an agent runs
size: M
requires: [F04]
novel: false
---

## What
Two protections for the terminal lifecycle. First, the app enforces a single
running instance — a second launch focuses the existing window rather than
starting a rival process. Second, closing a terminal tab is guarded: if Claude
is running in that terminal, the app refuses and tells the user to *"exit from
claude first"* rather than killing a live agent.

## Why it exists
Terminals hold running agents; an accidental kill loses work. Both mechanisms
protect that. The single-instance lock also prevents the subtler failure where a
second app instance fights over the same tmux sessions and tabs go missing — the
author explicitly chose this simple invariant over live reconciliation
machinery after losing tabs that way.

> *"prohibit closing session via 'x' if claude is running. Say to user 'exit from
> claude first'."*
> *"we should not allow second instance to run — only one instance. Remove
> reconciliation."*

## Acceptance criteria (EARS)
- When a second app instance is launched, the system shall not start; it shall
  focus the already-running instance's window instead.
- When the user attempts to close a terminal tab while Claude is running in it,
  the system shall block the close and instruct the user to exit Claude first.
- When no agent is running in a terminal, the user shall be able to close its tab
  normally.
- On detach (closing the view without closing the tab), the system shall keep the
  tab and its tmux session alive.

## Build notes
- Use the platform single-instance lock (e.g. Electron's `requestSingleInstanceLock`);
  on the second launch, focus the primary window and exit.
- Deliberately do **not** build live reconciliation of sessions across instances —
  the single-instance invariant is the fix, and reconciliation was removed as
  unnecessary machinery.
- Back the close-guard with an `isClaudeRunning` check on the terminal's session
  (inspect the running process in the tmux pane); only that state blocks the
  close.
