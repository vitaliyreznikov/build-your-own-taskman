---
id: O09
epic: O — Structured task documents (v2)
title: Per-subgoal terminals
size: M
requires: [O03, O06, F02, F04, F12]
novel: true
---

## What
Every subgoal row in the structured view (O06) carries its own terminal launcher.
Opening it creates a normal task terminal that is additionally **scoped to that
subgoal**: a terminal is linked to a task *and optionally a subgoal*, and each
subgoal row lists the terminals currently live on it — inline, with their agent
status icon — so the tree doubles as a map of what work is running where.

**Session naming.** A task terminal keeps its existing name: `task-<id>` for the
first, `task-<id>-<n>` for the rest. A subgoal terminal is
`task-<id>-sg-<slug>` for the first on that subgoal and
`task-<id>-sg-<slug>-<n>` after that. **Each subgoal has its own index space**, so
opening a second terminal on `g1.1` does not consume an index belonging to the task
or to a sibling subgoal.

**The slug is a hard constraint, not cosmetics.** Subgoal ids are dotted (`g1.1`).
tmux silently rewrites `.` to `_` inside a session name, and — worse — a session
whose name contains a dot cannot be addressed afterwards at all: `kill-session -t
task-I9-sg-g1.1` fails, because tmux parses the dot as a window/pane target. So
the id is **slugified into the name**: every character outside `[A-Za-z0-9_]`
becomes `_`. Slugification is lossy and not reversible, which is why the reverse
direction never tries to un-slug — the app maps a slug back to a real subgoal id by
reading the document, slugifying each of its subgoal ids, and comparing.

**The link is exported.** The session environment gets a subgoal variable
(`TASKS_SUBGOAL_ID`) alongside the existing `TASKS_TASK_ID` (F02), carrying the
**real** dotted id, so a lifecycle hook or any script inside the session knows
which subgoal it is working.

**The prompt is scoped.** Instead of F12's generic work-on-task prompt, a subgoal
terminal auto-starts the agent with a prompt that names the subgoal's id and text,
tells it to work *that* subgoal specifically, and tells it to keep that subgoal's
state and blockers current in the document. It keeps F12's status-marker
instruction verbatim and repeats the same warning that these files are shared and
must be re-read before writing.

**Status keying already generalizes.** Agent status is keyed per session name
(F05/H08), so a subgoal terminal reports its own status under its own name and gets
its own icon — on its subgoal row, and aggregated into the task card icon (H01)
with the task's other terminals.

Otherwise it is an ordinary task terminal: it survives app restarts, appears in the
terminal view and the tab strip (labelled with its subgoal rather than a bare
index), and is captured by the reboot-survivable snapshot (F14) so a restore
recreates it with its subgoal link intact.

## Why it exists
A large task's work is **parallel across its subgoals** — that is what the tree in
O03 is for. The terminal layer, though, only knew about tasks, so every agent on a
ten-subgoal task got the same brief: the whole document. That is the wrong brief in
both directions. It is too big (the agent reads phases it will not touch, and has
to decide for itself which part is its), and it is too vague (nothing says which
subgoal's state and blockers it is responsible for keeping honest). An agent scoped
to one subgoal gets a brief that is smaller, more accurate, and self-checking: it
knows exactly which node it must leave in a correct state.

The second half is the read. "What is actually in flight on this task" is currently
answered by counting tabs and remembering what each one was doing. With terminals
shown on the rows they belong to, the structured view answers it directly: subgoals
with live sessions have work happening, subgoals without do not, and the status icon
says whether that work is running, waiting, or needs the human. It is the same
parallel-supervision idea H08 pushed from task down to terminal, pushed one level
further — down to the unit of work the document already describes.

Read-only stays read-only (O06). Launching a terminal is not structured editing: the
human still does not edit subgoals in the app, they point an agent at one.

## Acceptance criteria (EARS)
- When the system renders a v2 task's subgoal tree, it shall offer a terminal
  launcher on every subgoal row.
- When the user launches a terminal from a subgoal row, the system shall create a
  task terminal linked to both that task and that subgoal.
- The system shall name the first terminal on a subgoal `task-<id>-sg-<slug>` and
  each subsequent one `task-<id>-sg-<slug>-<n>`, where the index space is per
  subgoal and independent of the task's own terminal indices and of any sibling
  subgoal's.
- The system shall slugify the subgoal id into the session name by replacing every
  character outside `[A-Za-z0-9_]` with `_`, so that no session name contains a
  character the terminal backend rewrites or cannot address.
- When the system needs the subgoal id for a given session name, it shall recover it
  by slugifying the ids in that task's document and matching, and shall not attempt
  to reverse the slug.
- If no subgoal id in the document slugifies to a session's slug, then the system
  shall treat that terminal as a plain task terminal rather than dropping it or
  showing it on an arbitrary row.
- When the system creates a subgoal terminal, it shall export the subgoal's real
  (unslugified) id into the session environment alongside the task id, before any
  user command runs.
- When a subgoal terminal auto-starts the agent, the system shall seed a prompt that
  names the subgoal's id and text, instructs the agent to work that subgoal, and
  instructs it to keep that subgoal's state and blockers current in the document,
  while retaining the status-marker contract and the shared-file re-read warning of
  the task prompt.
- When a subgoal terminal is resuming a saved session, the system shall not re-type
  the scoped prompt (per F12).
- While a subgoal has live terminals, the system shall list them on that subgoal's
  row as compact chips each carrying that terminal's own status icon, and shall
  focus a terminal when its chip is activated.
- While a subgoal has no live terminal, the system shall show only the launcher on
  that row.
- The system shall key a subgoal terminal's agent status on its own session name, so
  that its status is not merged with the task's other terminals' statuses, and shall
  still include it when aggregating a single icon onto the task card.
- The system shall label a subgoal terminal's tab with its subgoal rather than a
  bare terminal index.
- When the terminal snapshot is written, the system shall record a subgoal
  terminal's subgoal link, and when such a terminal is restored it shall come back
  under the same session name with the same subgoal link and environment.

## Build notes
- Widen the terminal record from `taskId` to `{ taskId, subgoalId? }` in the pty
  manager, the snapshot (F14), and the tab model. Everything that already keys on
  the session name (status H02/H08, session history I01, F09) needs no change —
  which is the point of putting the scope in the name.
- Keep two pure functions in the O01 document module so no caller invents its own
  rule: `slugifySubgoalId(id)` and `sessionNameFor(taskId, subgoalId?, index)`.
  Name allocation reads the live session list, filters by the subgoal's prefix, and
  takes max-index + 1 — do not derive the index from a count, which drifts after a
  kill.
- Prefix filtering must be exact-segment, not `startsWith`: `task-I9-sg-g1` is a
  prefix of `task-I9-sg-g11`. Compare the parsed slug, not the string head.
- Slug collisions are possible in principle (`g1.1` and `g1_1` both slugify to
  `g1_1`). Subgoal ids are allocated by the app (O03) — allocate so that the
  *slugified* form is unique within the document, and treat a collision found in an
  externally written document as the unmatched case above.
- The scoped prompt is F12's text with the target swapped: keep both prompts in one
  module and one status-marker string, so they cannot drift. The subgoal text can be
  long or contain newlines — truncate to a sane length and collapse whitespace
  before typing it into a shell.
- The shared-file warning matters more here than for a task terminal: several agents
  scoped to sibling subgoals write the *same* `I<N>.json`. Restate re-read-before-
  write in the prompt, and rely on atomic writes (A06) plus the file watcher (B11)
  rather than on any in-app lock.
- Chips on a subgoal row read the same keyed status entries as the tab icons (H08).
  Render them compactly — a deep tree with terminals on several nodes must not turn
  the row into two lines of controls.
- Killing a subgoal terminal is the ordinary F04 kill; nothing in the document is
  touched. The document is written only by the agent, never by terminal lifecycle.
- A subgoal terminal still triggers F09 (task to IN PROGRESS) — the task is the unit
  the board tracks, and there is no per-subgoal column.
