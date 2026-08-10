---
id: P03
epic: P — Agent-facing control API
title: "`create-task` command — an agent asks TaskMan to file a new task"
size: S
requires: [P01, A02, A07]
novel: false
---

## What
The second verb on the control API (P01): an agent asks the app to **create a new
task**, and — once the user approves — the app files it and returns the allocated
`I<N>`.

Deliberately tiny surface. CLI shape:

```
taskman create-task "<title>" [--body <text>]
```

- `action = "create-task"`, `params = { title, body? }`.
- `title` is the only required input. `body` is an optional first draft of the
  task note.
- Everything else a task can carry — priority, board, project, parent, column,
  document type (v1/v2), relations, autorun — is **intentionally not a parameter**.
  The task is created with the app's normal defaults, and any further shaping is
  done **after creation by editing the files** (`tasks.md` for the row fields,
  `I<N>.md` for the note), which an agent can already do directly. Keeping the verb
  to title(+body) is the whole point: it captures a task in one gesture without
  turning into a wrapper around every column of the index.

Approval sentence (P01's `describe`): *"Session in `<source.taskId>` wants to create
a task: "`<title>`""* — with *"(no caller)"* standing in when `source.taskId` is
absent.

On approve, the executor runs the **same create path the app's own quick-add uses**
(allocate the next free id, write the `tasks.md` row with defaults, seed the note,
place it in the ordering) entirely in the main process — no renderer round trip,
unlike P02. If a `body` was given, it is written into the new note. The request
resolves `done` with `result = { taskId, notePath }` so the caller knows the id and
exactly where to edit next. The board refreshes so the new card appears at once.

## Why it exists
Filing a task programmatically is the other half of "an agent acting on the board":
P02 lets an agent open work on a task; `create-task` lets it *capture* one. An agent
mid-flow often realises a follow-up belongs on the board — but it should not have to
hand-edit `tasks.md`/`order.md` (and risk racing the app's own writes) to add a row,
nor should it create work silently. `create-task` gives it a single approved call
that reuses the app's own, race-safe create path, then hands back the id so the
agent can flesh the note out by editing the file — which is exactly where all the
non-essential detail belongs.

## Acceptance criteria (EARS)
- The system shall register a `create-task` action taking a required `title` and an
  optional `body`, and shall reject a request whose `title` is missing or blank.
- When `create-task` is approved, the system shall create a task via the same
  create path as the app's quick-add (id allocation, index row with defaults,
  note seed, ordering), returning the allocated `I<N>`.
- When a `body` is provided, the system shall write it into the new task's note.
- When the request resolves `done`, its result shall carry the new task id and the
  path to its note file.
- The system shall refresh the board so the newly created task appears without a
  manual reload.
- The verb shall not accept parameters for priority, board, project, parent,
  column, type, relations, or autorun; those shall be left to post-creation file
  edits.

## Build notes
- Reuse the existing `createTaskInternal(partial)` (or equivalent quick-add path)
  with just `{ title }`; it already reads `tasks.md` late for race-safe id
  allocation and merge (A02/A07). Seed the body with the normal note-write function
  afterward when present.
- Executes fully in main (a file write), so — unlike P02 — it does not delegate to
  the renderer; it just pushes the app's existing "tasks changed" signal so the
  board repaints.
- Keep the parameter set frozen at title(+body). New knobs are a smell here: the
  design bet is that post-creation file editing covers them, and the CLI help / the
  seeded terminal prompt should say so explicitly.
