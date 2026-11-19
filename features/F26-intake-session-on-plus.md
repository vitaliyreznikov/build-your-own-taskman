---
id: F26
epic: F — In-app terminals
title: "+" opens a short-lived intake session (per-category prompt, memory of past intakes, hands off to a task worker)
size: M
requires: [F13, B18, P02, P03, F06]
novel: true
---

## What
The Terminal-strip **"+"** stops creating a task. After the category picker (B18)
it opens an **intake terminal**: a short-lived agent session that is *not* bound to
any task, whose only job is to find out what the user wants, do the minimum
research, and route the work.

1. **No placeholder.** "+" reserves no id and writes no row/note (it supersedes
   F13's reserve-then-fill flow). The terminal is named
   `intake-<category>-<YYYYMMDD-HHmm>`, exports no task id, joins no board, and
   its tab is inserted at the **left** end of the strip (every other new tab
   still appends on the right).
2. **Per-category prompt.** The agent is started with a prompt assembled from two
   plain files in the data root: `intake/common.md` (the shared procedure) +
   `intake/<category>.md` (what is specific to bolt / brootto / personal / …).
   A missing category file is just skipped. Both are git-synced like every other
   data file.
3. **Memory of past intakes.** Each intake writes its own result file
   `intake/sessions/<category>/<YYYYMMDD-HHmm>.md`, which starts with a short
   `## Summary` (what was asked, what was decided — new `I<N>`, picked `I<M>`, or
   answered with no task). At startup the agent reads the summaries of the
   **last 10** session files of its category (newest first) and reads further
   back, or whole files, only when it needs to.
4. **The menu.** The agent opens by asking what the user wants — typically: a new
   task, continue an existing task, or a quick question that needs no task. It
   does basic research (index, notes/documents, KB), checks for an existing task
   before filing a new one, and confirms its routing choice with the user.
5. **Hand-off, not take-over.** When the work is routed to a task — one it just
   filed (`taskman create-task` + file edits) or an existing one — the intake
   launches it as a **task worker** (`taskman run <id> [--goal <sg>]`, P02,
   user-approved) and then finishes: it writes its session file and ends with a
   `done` marker. The intake never does the task's work itself.
6. **A second terminal on demand.** `taskman run` gains `--new`: open an
   *additional* terminal for the task (or subgoal) even if one is already open,
   instead of focusing/arming the existing one. The intake uses it when the user
   wants a parallel session on a task that is already running.
7. **Self-editing prompt.** When the user asks to change how future intakes
   behave ("remember…", "from now on…", "change the prompt"), the agent edits
   `intake/<category>.md` (or `intake/common.md` for a change meant for every
   category) and records the edit in its session file. It doesn't edit the prompt
   on its own initiative.
8. **A finished intake gets out of the way.** When the intake agent ends a turn
   with `✅ done` (hand-off made, session file written), its tab is marked
   finished at once (dimmed, ✓), and **2 minutes later the tab closes itself**
   (the tmux session is killed). Any sign of life in between cancels the close:
   the agent goes back to `working`/`attention`/`waiting`, or the user types in
   that terminal. A `waiting` or `attention` intake is never closed
   automatically.

## Why it exists
F13 made "+" file a task before anyone knew what the work was. In practice "+"
is pressed for several different reasons: start something new, pick up
something old, ask a quick question. A reserved placeholder is wrong for the last
two, and every abandoned "+" left a `New task…` card behind. A dedicated intake
step routes the request first and creates a task only when one is needed.

Different categories need different first questions (a bolt request wants the
PR/alert/service; a brootto one wants the client/campaign), so the prompt is
per-category, and because it's a plain file, the user can teach it in plain
words. The session files give the intake continuity ("same thing as yesterday's
request") without a database. Handing off to a task worker instead of working in
place keeps every task's work on its own terminal, card and timeline.

## Acceptance criteria (EARS)
- When the user confirms a category in the Terminal-strip "+" picker, the system
  shall open an intake terminal for that category without creating a task row,
  task note, or reserved id.
- When the user cancels the picker, the system shall open no terminal.
- The system shall insert the intake terminal's tab at the left end of the
  terminal strip and focus it.
- The intake terminal shall not export a task id and shall not appear on any
  board.
- When the intake terminal starts, the system shall auto-start the agent with a
  prompt composed of `intake/common.md` followed by `intake/<category>.md`,
  skipping a file that doesn't exist, plus the session-file path for this run and
  the status-marker contract.
- The prompt shall instruct the agent to read the summaries of the 10 most recent
  session files of its category before asking anything, and allow it to read
  older ones when needed.
- The prompt shall instruct the agent to ask the user what they want, check
  existing tasks before filing a new one, and launch the chosen task through the
  control API (`taskman run`) rather than working on it in the intake session.
- When an intake session ends, the agent shall have written
  `intake/sessions/<category>/<stamp>.md` beginning with a `## Summary` section
  that names the outcome (task id created/launched, or no task).
- When the user asks to change future intake behavior, the agent shall edit
  `intake/<category>.md` or `intake/common.md` accordingly and note the change
  in its session file.
- When `taskman run <taskId> --new` is approved, the system shall open an
  additional terminal for that task (or subgoal, with `--goal`) even if one is
  already open; without `--new` the existing focus-or-open behavior shall be
  unchanged.
- When an intake terminal's agent status becomes `done`, the system shall mark
  that tab as finished (visually distinct from live tabs) immediately.
- While an intake terminal stays `done` with no keyboard input from the user,
  the system shall close the tab and end its terminal session 2 minutes after
  the status became `done`.
- If the intake's status changes away from `done`, or the user types into that
  terminal, before the 2 minutes elapse, the system shall cancel the pending
  close and remove the finished mark.
- The system shall never auto-close an intake whose status is `waiting` or
  `attention`, nor any non-intake terminal.
- The board-view new-task bar (B17) shall keep creating tasks directly, with no
  behavior change.

## Build notes
- Prompt assembly happens in main at launch time: read both files fresh each
  time (no caching), so an edit made by the previous intake applies to the next
  one immediately.
- The intake terminal is a plain non-task terminal with an auto-start prompt.
  Reuse the auto-start path with an `intake: {category}` option in place of F13's
  `createTask` flag, and export `TASKS_INTAKE_CATEGORY` and `TASKS_INTAKE_FILE`
  so the agent (and hooks) know where to write.
- Status markers still drive the tab icon. Since there's no task, the status is
  keyed on the terminal name only and no card shows it.
- Left insertion only applies to the intake tab: prepend it to the terminal list
  instead of appending. Persisted tab order (F07) takes it from there.
- `--new` maps to the store's existing "new numbered terminal for a task"
  (`task-<id>-<n>`) and to the subgoal opener's `fresh` flag, so no new terminal
  kind is needed. Only the API verb and CLI change.
- Seed `intake/common.md` with the procedure (menu, research, dedup, create →
  edit files, launch via `taskman run`, write the session file, prompt
  self-editing rules) and one `intake/<category>.md` per category in the
  categories file. Keep the files short: they are prompts.
- The session-file stamp is minute-resolution. If two intakes start in the same
  minute, add a `-2` suffix rather than overwriting.
- Auto-close lives in the renderer: watch the per-terminal status (H08) for
  `intake-*` tabs, start a 2-minute timer on `done`, and close the tab through
  the normal close path (the F10 "agent is running" guard doesn't apply; the
  agent is idle at its prompt). Any status change or terminal
  input cancels the timer and drops the mark; a later fresh `done` starts a new
  2 minutes. Timers don't survive an app restart: on startup, an intake
  that is already `done` gets a fresh 2 minutes.
- The F13 create-task prompt can stay in code for a direct "new task + agent"
  path, but "+" no longer uses it.
