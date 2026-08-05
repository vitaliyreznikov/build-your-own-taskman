---
id: F13
epic: F — In-app terminals
title: "+" new-terminal button reserves a task id and spawns an agent to fill+solve it
size: M
requires: [F06, F12, F09, A07]
novel: true
---

## What
The "+" button in the terminal tab bar no longer opens a bare shell. Clicking it:

1. **Reserves a task id up front** — allocates the next free `I<N>` (A07) and
   creates a *placeholder* task (index row + note file, title `New task…`). This
   is what "reserves" the id: because allocation unions `tasks.md` rows and note
   files, a committed placeholder guarantees no later allocation reuses `I<N>`.
2. **Opens a task-linked terminal named for that id** (`task-I<N>`), so the tab
   reads **`I<N>` from the start** and never has to be renamed once the real task
   lands. Being a task terminal, it exports `TASKS_TASK_ID` (F02) and joins the
   IN PROGRESS board on open (F09) like any other task terminal.
3. **Auto-starts the agent with a create-and-solve prompt** (built on F12) that
   *names the reserved id*: the agent interviews the user for a title and short
   description, **checks whether an existing task already covers this work**, and
   — if none does — fills the placeholder in (real title in `tasks.md`, contents
   in `I<N>.md`) per `AGENTS.md`, then immediately starts working on it — all in
   the one session. The prompt also restates the status-marker contract so the
   supervision layer (H03/H04) drives the tab/card icon while the work runs.

The dedup step is the interesting half. "A new task" is the wrong answer often
enough — what the user just described is frequently the next move on something
already on the board — so the prompt makes the agent *look* before it files:
search the index and the notes it points at, name the candidate, and ask. If the
user says continue there instead, the reserved placeholder has to go, and the
prompt spells out how, because a terminal agent cannot call the app's delete
path: drop the `I<N>` row from the index, its id from the ordering file and the
board files, any relation rows naming it, and the note file itself. The session
then carries on against the existing task in the same terminal — whose tab keeps
the now-dead reserved id, so the prompt also tells the agent to send the user to
that task's own card if they want the status icon to land on the right task.

The tab title concern ("how will it update when the task is created?") is
designed away: the id is fixed before the terminal exists, so the tab is correct
from frame one. Only the *card* title changes, when the agent replaces the
`New task…` placeholder.

### The exact prompt (from the reference implementation)
After typing `claude` and waiting until it looks ready, the app types this
verbatim (`<N>` is the reserved id):

```
Task I<N> was just created in TaskMan (this repo) as a placeholder and this terminal is linked to it. Ask the user for a few details — enough for a clear title and a short description. Then check the existing tasks before filing a new one: search tasks.md and the notes/documents it points at for work this belongs to, and if an open task already covers it, name that task and ask whether to continue under it instead. If the user approves, delete this placeholder — remove the I<N> row from tasks.md, its id from order.md and from views.md and views/*.md, drop any relations.md rows mentioning it, and delete I<N>.md — then carry on in this same session against the existing task (its terminal tab keeps the I<N> label, so tell the user to open a terminal from that task's card if they want its status icon to follow the work). Otherwise fill the task in: set its title in tasks.md and write its I<N>.md note, following the conventions in AGENTS.md. Then immediately start working on it in this same session. Report status inside your answers: when your phase changes add a line "🤖 working: <what you're doing>", and end every turn's final message with one marker line — "✅ done: <one-line summary>", "⏳ waiting: <external process: job, CI, review — never waiting on me>" or "⚠️ attention: <anything you need from me>" (a question = attention, not done).
```

(The file names in the delete recipe are this feature's own persistence files —
index, ordering, boards, relations — so a port names whatever it calls those. If
per-task document types (O02) are built, that epic adds one more clause to this
prompt: asking which format the new task should use.)

## Why it exists
Capturing a new task and starting on it were two separate motions: hit a quick-add
bar (B04) to file the task, then open its terminal (F12) to point an agent at it.
The "+" button — already the most reachable control on the terminal view —
collapses both into one gesture. Reserving the id *before* launching the agent is
what makes it clean: the terminal is a first-class task terminal immediately
(stable tab label, `TASKS_TASK_ID`, board membership, per-terminal status keying),
instead of a nameless `term-<n>` that would have to be re-identified after the
agent got around to creating the task.

## Acceptance criteria (EARS)
- When the user clicks the new-terminal ("+") button, the system shall reserve the
  next free task id by creating a placeholder task (index row + note) before
  opening the terminal.
- The system shall open a task-linked terminal named for the reserved id so its
  tab shows `I<N>` from the moment it appears, with no later rename.
- When the terminal opens, the system shall auto-start the agent and type a prompt
  that names the reserved id and instructs the agent to elicit a title and
  description, fill the placeholder in per the agent guide, then start solving it.
- The prompt shall instruct the agent to search the existing tasks for one the
  described work belongs under, and — where one is found — to name it and ask the
  user whether to continue under it rather than filing the new task.
- When the user approves continuing under an existing task, the prompt shall
  instruct the agent to delete the reserved placeholder from every place it was
  written (index row, ordering, board memberships, relations, note file) and to
  continue against the existing task in the same session.
- The prompt shall restate the status-marker contract so the agent emits the
  markers the supervision layer reads.
- The reserved id shall not be handed out to any subsequent allocation, even
  before the agent has written the real title/note.
- While auto-start's readiness/timeout handling (F12) shall apply unchanged, the
  create-task prompt shall be selected (instead of the plain work-on-task prompt)
  for a terminal opened via this button.

## Build notes
- Reservation reuses the normal task-create path (allocate id → write `tasks.md`
  row → `ensureNote` → order), so no new persistence format is introduced; the
  placeholder is just a task whose title the agent overwrites. Because id
  allocation already unions rows *and* note files (A07), a committed placeholder
  is a durable reservation.
- The only additions over F12 are: (a) a `createTask` flag on the terminal-open
  request, (b) relaxing the auto-start gate to fire on `autoClaude || resume` when
  `taskId || createTask` (it previously required a `taskId`), and (c) selecting
  the create-task prompt — which still references the (now reserved) `taskId` — in
  place of the work-on-task prompt.
- The button handler is the only caller that sets `createTask`; every other
  auto-start path passes a `taskId` and gets the work-on-task prompt unchanged.
- Creating the placeholder is a self-write, so suppress the file-watcher echo
  (no "changed on disk" banner) and refresh the board locally so the new card
  appears at once.
- Keep the seeded status-marker text in sync with the agent guide, same as F12.
- The delete recipe in the prompt has to be written out longhand and kept in step
  with the app's own delete handler, because the agent is a terminal process with
  no access to that code path — it edits the files directly. Miss the ordering or
  board files and the deleted id leaves dangling references; miss the note file
  and id allocation (A07) keeps the id reserved forever.
- Deleting the placeholder does *not* re-key the terminal: `TASKS_TASK_ID` is
  fixed at spawn, so per-terminal status (H08) keeps reporting under the dead id.
  Telling the user to open a terminal from the surviving task's card is the cheap
  fix; re-keying a live terminal would be a separate feature.
- Tradeoff: clicking "+" and then abandoning leaves a `New task…` placeholder on
  the board. That is the intended "create it at the same time" semantics; the
  agent renames it within its first turn — or removes it, if the work turned out
  to belong to a task that already exists.
