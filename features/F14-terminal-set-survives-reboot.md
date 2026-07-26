---
id: F14
epic: F — In-app terminals
title: Terminal set survives a reboot — auto-saved snapshot + one-click restore
size: M
requires: [F06, F07, F09, I02]
novel: true
---

## What
Make the set of open terminals survive a **machine reboot**, not just an app
restart.

The tmux backend (F01) already makes tabs survive quitting, crashing and
reloading the app: tab membership is rediscovered from `tmux list-sessions`, so
the sessions — with their processes, scrollback and running agents — are simply
re-attached. A reboot kills the tmux server, so `list-sessions` comes back empty
and **every tab is gone**, with no record that they ever existed.

Two halves:

1. **Auto-saved snapshot.** The app continuously records the current terminal set
   to a machine-local `.terminals.json`: for each session its name, task id,
   working directory, left-to-right position, and the Claude session a restore
   should resume (its id *and* the directory it ran in). This file is also the
   single home of the **tab order** — one file describes the terminal set
   completely, so nothing can restore the tabs but lose their arrangement. It is written on every change that matters (a terminal
   opened, a tab closed, tabs reordered), on a slow heartbeat while terminals are
   open, and on app quit / system shutdown. The user never has to remember to save
   — a kernel panic at 3am must not cost the day's terminals.
2. **Explicit restore.** On startup the app compares the snapshot with the live
   tmux sessions. When the snapshot names terminals that no longer exist, the tab
   bar offers **`↩ Restore N`**. Pressing it recreates each missing session — same
   session name (so the tab label, `TASKS_TERM_ID` status key and saved order all
   line up), same working directory — and re-launches Claude with
   `claude --resume <session id>` (I02) so each terminal continues its
   conversation instead of starting cold.

Restore is deliberately **not** automatic: it starts N agent processes, and which
of yesterday's fifteen terminals the user actually wants back is a judgement call.
Saving is automatic because the opposite is true — there is no judgement in it,
and the moment you need the data you have already lost the chance to ask.

A `💾 saved HH:mm` indicator sits next to the restore button; it shows when the
snapshot was last written and forces a save on click. Its real job is to make the
automatic saving *visible* — a silent safety net is indistinguishable from a
broken one.

## Why it exists
The terminal set is the working state of the whole app: fifteen tabs, each an
agent mid-task, is what "where I was" means here. tmux made that survive
everything except the one event that happens on purpose every few weeks — an OS
update reboot. The cost of the gap is not the shells (cheap) but the reconstruction:
remembering which fifteen tasks were live, reopening them, and re-establishing each
agent's context.

The pieces for the fix already exist and are only unconnected: per-task Claude
session ids are already recorded (I01), `claude --resume` is already wired into
terminal open (I02), tab order is already persisted, and session names are already
deterministic from task id + index. The single missing artifact is a durable record
of *which sessions existed*, because that has always been read out of a live tmux
server's RAM.

Recording it also removes the last reason the app cannot be restarted freely: it
turns "don't reboot, you'll lose the terminals" into a button.

## Acceptance criteria (EARS)
- The system shall maintain a machine-local snapshot of the open terminals holding,
  per terminal, its session name, task id (when it is a task terminal), working
  directory, left-to-right position, and the Claude session a restore should resume.
- The snapshot shall be the single source of the terminal tab order, and the app
  shall order the tabs by it on startup.
- Because agent session history is keyed by working directory, the system shall
  record the directory the resumable session ran in, and shall start a restored
  terminal in that directory so the resume finds the conversation.
- Where the directory a recorded session ran in no longer exists, the system shall
  not offer that session for resume, and where a restore target directory has
  disappeared it shall fall back to the data root rather than fail to open the
  terminal.
- The system shall update the snapshot when a terminal is opened, when a tab is
  closed, and when tabs are reordered, without any user action.
- While terminals are open, the system shall refresh the snapshot periodically so
  that a working-directory change or an externally created session is captured.
- When the app quits, or the operating system reports an impending shutdown, the
  system shall write the snapshot before exiting.
- Where the snapshot names terminals that no longer exist in the terminal backend,
  the system shall offer a restore action stating how many terminals it would bring
  back, and shall not restore them automatically.
- When the user invokes restore, the system shall recreate each missing terminal
  under its original session name and working directory, in the recorded order, and
  shall re-launch the agent with the recorded Claude session id so the conversation
  resumes.
- Where a missing terminal has no recorded Claude session id, the system shall
  restore it as a plain terminal rather than skipping it.
- When restoring, the system shall not re-add the terminal's task to the IN PROGRESS
  board (F09), because restoring is resuming, not starting new work.
- The system shall display when the snapshot was last saved, and shall save on
  demand when that indicator is activated.
- The system shall never let snapshotting or restoring kill, replace or disturb a
  terminal that is currently alive.

## Build notes
### Storage
`.terminals.json` in the data root next to `.llm-status.json` / `.claude-sessions.json`
(gitignored, machine-local — session names are meaningless on another machine).
Shape: `{ savedAt, terminals: [{ name, taskId, cwd, order, claudeSessionId, lastSeenAt }] }`.
Write through the existing atomic-write helper; a torn snapshot is worse than a
stale one.

### Who owns what
The main process owns the *contents*: it can enumerate real sessions
(`list-sessions`), read each pane's cwd (`display-message -p '#{pane_current_path}'`)
and look up the latest session id per task in `.claude-sessions.json`. The renderer
owns only the *order* (it holds the tab array), so the save IPC takes an order array
and the main process derives everything else. Main caches the last order it was
given so it can still save on `will-quit` / `powerMonitor('shutdown')`, when no
renderer round-trip is possible.

### Picking what to resume
A task accumulates sessions from several directories (the data root, a worktree, a
temp scratchpad), and Claude keys its history by cwd
(`~/.claude/projects/<slugified-cwd>`) — so `claude --resume <id>` silently finds
nothing when the terminal starts elsewhere. Prefer the newest session whose cwd is
the pane's current cwd (resume works *and* the terminal returns to where it was);
otherwise the newest session whose directory still exists; otherwise no resume at
all. Never offer a session from a directory that is gone — a deleted worktree or a
wiped `/private/tmp` scratchpad would make the restore fail outright.

### Tab order
Order used to live in renderer localStorage, written on drag-reorder. It moves into
the snapshot: reordering saves the snapshot, and startup reads the snapshot before
seeding the tabs (a late-arriving order would leave the first render mis-ordered).
The old localStorage key is read once as a migration seed and never written again.

### Restore path
Reuse `terminal:open` — it already accepts an explicit session `name` and a
`resumeSessionId` (I02). Two small additions: an optional `cwd` (the session
creator currently hard-codes the data root), and a `restore` flag that suppresses
the F09 IN PROGRESS write. Restore sequentially, in recorded order, so names and
tab positions land deterministically and fifteen `claude --resume` launches don't
stampede. Sessions that are already alive are skipped by name — the existing
"attach, don't recreate" behaviour of `openSession` already guarantees this.

### Stale status
`.llm-status.json` is keyed by session name and survives reboots, so a restored
tab would otherwise wear a badge from days ago. Prune entries whose session no
longer exists when the snapshot is reconciled at startup.

### Caveats
This restores the *terminal set*, not the processes: an agent that was mid-turn
comes back at the start of a resumed conversation, and scrollback (which lives in
the tmux server) is lost. Capturing scrollback to disk (`pipe-pane`) and replaying
the tail on restore is a separate, larger feature; the resumed transcript is a
better carrier of context than a screenful of text anyway.
