---
id: O02
epic: O — Structured task documents (v2)
title: Per-task document type + v1/v2 coexistence
size: M
requires: [O01, A02]
novel: false
---

## What
A new **`Type` column** in the central index (`tasks.md`) records which detail
format each task uses:

| cell | meaning |
| --- | --- |
| empty or `v1` | Markdown note — `I<N>.md` (A01) |
| `v2` | structured JSON document — `I<N>.json` (O01) |

The type is chosen **at creation, per task**, and both formats **coexist
forever**. There is no flag day, no global migration, no "v2 mode" for the app. A
board can hold v1 and v2 tasks side by side; each task's own row decides how the
app reads, renders and writes its detail.

A missing cell reads as `v1`, so every existing row keeps working untouched.

## Why it exists
The structured document is a better fit for long multi-phase work and a worse fit
for a two-line reminder. Forcing one format on every task would either burden
trivial tasks with a schema or hold back the tasks that need structure — so the
choice belongs to the task, not the app.

Coexistence also has to be *permanent* rather than transitional: a spec that
promises to delete v1 later means every v1 task is technical debt, and hundreds of
existing notes are the app's actual memory. Making the type a normal index column
turns "which format" into ordinary data — greppable, git-diffable, editable by an
agent — rather than a build-time decision.

The type is authoritative **on purpose**. The tempting shortcut is to sniff the
filesystem: if `I<N>.json` exists it's v2, else v1. That is wrong in every partial
state that actually occurs — a half-finished conversion, a leftover file from a
reverted conversion, an agent that wrote the JSON before updating the row — and the
failure mode is silent: the app renders one file while the human and the agent are
editing the other.

## Acceptance criteria (EARS)
- The index (`tasks.md`) shall carry a `Type` column whose value is empty, `v1`, or
  `v2`.
- When the `Type` cell is empty or absent, the system shall treat the task as `v1`.
- When the system opens a task, it shall choose the detail reader and renderer from
  the task's `Type` cell and shall not infer the type from which detail file exists
  on disk.
- When a task is created, the system shall let the creator choose the type and shall
  write the chosen value into the row.
- When the type is `v2` but `I<N>.json` is missing or unparseable, the system shall
  show an error state for that task rather than silently falling back to the
  Markdown note.
- When the system allocates a new task id, it shall derive the maximum from both
  `I<N>.md` and `I<N>.json` files on disk in addition to the index rows.
- The system shall render v1 and v2 tasks together in the same board, columns,
  ordering, search and relations without either format degrading the other.
- When an unknown value appears in the `Type` cell, the system shall treat the task
  as `v1` and shall not drop the row.

## Build notes
- Extend the index parser/serializer (A02) with one optional column. Round-trip
  fidelity matters: a v1 row that never had the cell should not gain noise, and an
  unrecognized column must survive a rewrite.
- **Id allocation is the sharp edge.** A07 already unions index rows with `I<n>.md`
  files on disk; extend that scan to `I<n>.json`. Miss it and the sequence is: an
  agent creates task `I400` as a v2 document, the app allocates `I400` again for
  the next task, and the agent's document is overwritten. Same non-atomicity as
  A07 (note write and row write are two operations), one more extension to glob.
- Thread the type through as an explicit parameter, not an ambient mode. One
  helper — `detailKind(taskId): 'v1' | 'v2'` reading the row — and every consumer
  (detail panel, search, watcher, index cache, conversion) branches on its result.
- Creation paths that must offer the type: the quick new-task bar (B04) and the
  agent-driven "+" terminal (F13). For F13 the reserved row is written before the
  agent runs, so the type must be decided at reservation time and stated in the
  agent's prompt — otherwise the agent guesses which file to write.
- Deleting a task must delete whichever detail file the row points at *and* sweep
  the other extension, so a reverted conversion never leaves an orphan that a
  future filesystem-sniffing bug can resurrect.
- Keep the column narrow and human-typable. It is expected to be edited by hand and
  by agents; `v2` is three characters for that reason.
