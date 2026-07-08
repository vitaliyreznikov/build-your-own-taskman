---
id: F01
epic: F — In-app terminals
title: Embedded tmux-backed terminal per task
size: L
requires: [B12]
novel: true
---

## What
A real, interactive shell terminal running *inside* the task app. Each task can
open a terminal (an `xterm.js` view in the renderer) backed by a persistent
`tmux` session on the host, one session per task (e.g. `task-I<N>`). Because it's
tmux-backed, the session survives the terminal view being closed and reopened, and
survives app restarts.

## Why it exists
The core workflow is "point an agent at a task and let it work." The agent should
run **where the task lives** — same window, same context — not in a separate
terminal app you alt-tab to. Putting the terminal inside the task is also the
*enabling* decision for the entire live-status supervision layer: the session
carries the task identity (see F02), which is what later lets Claude Code hooks
report per-task, per-terminal status back onto the board.

> This is the second of the two ideas the author considers genuinely fresh:
> *"run terminal inside task app."*

## Acceptance criteria (EARS)
- When the user opens a terminal on a task, the system shall attach an interactive
  shell rendered inside the app (full keyboard input, ANSI colors, resize).
- The system shall back each task's terminal with a persistent tmux session keyed
  to the task, so that when the user closes and reopens the terminal view, the
  same session (with its running processes and scrollback) is reattached.
- When the app restarts, the system shall reattach to any still-living tmux
  sessions rather than spawning fresh ones.
- The user shall be able to select terminal output and copy it to the clipboard.
- The terminal shall correctly handle a fresh xterm instance re-attaching to a
  live session (repaint), and shall not display a stale session after switching
  tasks.

## Tools / libraries (the concrete stack)
The terminal is the one feature where the specific tools matter — here's exactly
what the reference app uses. Substitute equivalents if your stack differs, but
this combination is what makes persistence + reattach work:

- **`@xterm/xterm`** — the terminal emulator rendered in the app (renderer process).
- **`@xterm/addon-fit`** — resizes the xterm grid to the container on layout change.
- **`node-pty`** — spawns a pseudo-terminal from the Electron **main** process.
  Native module: rebuild it against Electron (`electron-rebuild -f -w node-pty`)
  in a postinstall step, or it won't load.
- **`tmux`** — the persistence layer. **node-pty does not spawn your shell
  directly — it spawns `tmux`**, one session per terminal, each on a *dedicated
  tmux socket*. That's the whole trick: the shell + running agent live in tmux, so
  closing/reopening the view or restarting the app just re-attaches to the same
  session. Scrollback lives in tmux (mouse wheel → copy-mode), so keep xterm's own
  scrollback small.
- **IPC** (Electron `ipcMain`/`ipcRenderer` + preload) pipes pty output → renderer
  and keystrokes → pty.

## Build notes
- Architecture: renderer holds the `@xterm/xterm` view; the Electron main process
  owns a pty manager that spawns/attaches tmux via `node-pty` and pipes data over
  IPC. Keep the pty lifecycle in main so it survives renderer reloads.
- Keep xterm instances in a **module-level registry keyed by session** so they
  survive React remounts — don't recreate the Terminal on every render, or you
  lose scrollback and the attach.
- Real edge cases the reference implementation hit (honor these):
  - **Repaint on re-attach:** when a fresh xterm re-attaches to an existing tmux
    session, force a repaint or the pane renders blank.
  - **Evict stale element on session switch:** when a reused host view switches to
    a different session, drop the old element so you don't show the wrong terminal.
  - **Cold-socket race:** guard against attaching before the tmux socket is ready.
- Export the task id into the session's environment here or in F02
  (`TASKS_TASK_ID`) — downstream features (H03 hooks, F09 auto-IN-PROGRESS,
  I01 session history) depend on it.
- This is the single hardest feature in the catalog and the gate for epics G, H,
  and I. Budget accordingly; get reattach rock-solid before building on it.
