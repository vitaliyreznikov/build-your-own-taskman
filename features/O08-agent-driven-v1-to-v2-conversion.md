---
id: O08
epic: O — Structured task documents (v2)
title: Agent-driven v1→v2 conversion
size: S
requires: [O01, O02, O04, O05, F12]
novel: false
---

## What
Converting an existing Markdown task to a v2 document is done **by an agent, in the
task's own terminal** — not by a transform in the app. A **"Convert to v2"** control
on a v1 task opens (or reuses) that task's terminal via the F12 auto-start path and
seeds a **conversion prompt** instead of the ordinary work-on-task prompt. The prompt
tells the agent to:

1. Read `I<N>.md`.
2. Split it into **`body_human`** (what should be done — intent, definition of done,
   constraints) and **`body_state`** (where this stands now, written for a human
   reading it cold).
3. Lift checklists and phase lists into **nested subgoals** with states, and record
   any known blockers as blocker records (O03).
4. Lift reusable question→answer findings into **`I<N>/kb.jsonl`** (O04).
5. Lift dated status bullets into **`I<N>/timeline.jsonl`**, oldest first (O05).
6. **Preserve the original note** under the task's detail directory (A08), e.g.
   `I<N>/note-v1.md`.
7. Write `I<N>.json` and set the task's `Type` cell to `v2` (O02).
8. Report what it did and what it was unsure about.

**Nothing is deleted until the human has seen the result.** The original note is kept,
and the human reviews the converted document in the structured view (O06) before any
cleanup.

## Why it exists
Conversion looks mechanical and isn't. Every step above is a judgment call: which
sentences are *intent* versus *current state*, which bullets are still true, which
findings are reusable facts versus one-off noise, which nesting reflects how the work
actually decomposes. A deterministic transform can only do the shape — dump the whole
note into one body and produce a v2 document with none of the value of a v2 document.
Worse, it would look successful.

An agent, on the other hand, is *good* at exactly this reading task, and the task's
terminal is where it already has the task's context. So the app's job is not to
convert; it is to **put the agent in front of the task with the right instructions**.
This is the same pattern as F13 (the "+" button spawns an agent to fill in a task)
rather than a form.

Keeping the original note is the cheap insurance that makes the whole thing safe to
try: if the split is wrong, nothing was lost, and the human can re-run the conversion
or fix it in an editor. Irreversible automated restructuring of the app's memory is
not a trade worth making to save a file.

## Acceptance criteria (EARS)
- When the user invokes convert-to-v2 on a v1 task, the system shall open that task's
  terminal and seed a conversion prompt naming the task's note file.
- The conversion prompt shall instruct the agent to split the note into `body_human`
  and `body_state`, lift checklists into nested subgoals, lift reusable findings into
  the task KB, and lift dated status bullets into the timeline.
- The conversion prompt shall instruct the agent to preserve the original note under
  the task's detail directory and to set the task's `Type` cell to `v2`.
- The system shall not itself parse, transform, or rewrite the note's content as part
  of conversion.
- The system shall not delete the original note as part of conversion.
- When the agent has written `I<N>.json` and set the type, the system shall render the
  task in the structured view on the next read.
- While the type cell still says `v1`, the system shall keep rendering the Markdown
  note even if `I<N>.json` already exists.
- When the conversion is offered on a task that is already `v2`, the system shall not
  offer it or shall refuse it.
- If the agent's document fails to parse, then the system shall show the parse error
  state (O06) with the raw text, leaving the preserved note intact.

## Build notes
- Implementation is small on purpose: reuse `termOpen({ taskId, autoClaude: true })`
  with an `extraPrompt` (the same seam L01 uses for the review prompt) carrying the
  conversion instructions in a single submission. No new terminal plumbing.
- Keep the conversion prompt in the prompt-parts library (G01) rather than hard-coding
  it, so it can be edited without a rebuild and re-run by hand on a task the app isn't
  driving.
- Order of the agent's writes matters for the reader: write `I<N>.json` and the side
  files **first**, flip the `Type` cell **last**. Because the type is authoritative
  (O02) and never inferred from the filesystem, that order means the task is either
  fully v1 or fully v2 at every instant — there is no window where the app looks for a
  document that isn't there yet.
- Tell the agent the id is already allocated and must not change, and that ids inside
  the document (subgoals, KB entries) are new and stable from now on (O03/O04).
- Batch conversion is deliberately not a feature. Conversion is per task, initiated by
  the human, reviewed by the human. A "convert everything" button would produce a
  hundred unreviewed documents, which is the failure mode this design exists to avoid.
- Cleanup (deleting the preserved `note-v1.md`) is a separate, manual act. Don't
  schedule it, don't offer it in the same click.
