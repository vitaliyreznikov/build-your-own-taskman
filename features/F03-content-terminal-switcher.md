---
id: F03
epic: F — In-app terminals
title: Content/terminal switcher on a task
size: S
requires: [F01]
novel: false
---

## What
A toggle inside a task's panel that switches its main area between the task's
Markdown note (content) and its embedded terminal. One task, one panel, two
views you flip between — you read/edit the note, then flip to the shell where an
agent works on it.

## Why it exists
The whole point of F01 is that the agent runs *where the task lives*. The
switcher is what makes "same window, same context" concrete: the note (the
prompt) and the terminal (the agent working the prompt) are two faces of the
same task, not two windows to alt-tab between.

## Acceptance criteria (EARS)
- When viewing a task, the user shall be able to switch the panel between the
  note content view and the terminal view.
- When the user switches to the terminal view, the system shall attach the
  task's terminal (creating it on first use, reattaching thereafter per F01).
- When the user switches back to content, the system shall render the task's
  Markdown note without tearing down the underlying tmux session.
- The system shall preserve each view's state across switches (scroll position /
  live terminal), so flipping back and forth does not reset the session.

## Build notes
- Toggling to content must not kill the pty — the session lives in main and
  outlives the view being hidden (F01). Hide/detach the xterm element, don't
  destroy the session.
- Honor F01's reattach edge cases when flipping back to the terminal: force a
  repaint on re-show, and if the host view is reused for a different task's
  session, evict the stale element first so you never show the wrong terminal.
