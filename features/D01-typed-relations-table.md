---
id: D01
epic: D — Task relations
title: Typed relations table (relations.md)
size: M
requires: [A02]
novel: false
---

## What
Task-to-task relationships live in a single Markdown table, `relations.md`, with
columns `From | Type | To`. Each row is one directed, typed link between two task
ids — for example `blocks` or `spawned`. Like the rest of the model, relations
are flat, human- and agent-legible text, not database rows.

## Why it exists
Tasks are not islands: one blocks another, one spawns its subtasks. Capturing
those links in a plain table keeps the relationship graph in the same AI-first,
file-as-source-of-truth form as everything else, so an agent reading the repo can
see how tasks connect.

## Acceptance criteria (EARS)
- The system shall store all task relations as rows of a `From | Type | To` table
  in `relations.md`.
- Each relation shall reference two task ids and one relation type.
- When the user adds or removes a relation, the system shall update `relations.md`
  accordingly (see D02).
- When the app reads relations, it shall resolve the `From`/`To` ids against the
  global task index (A02).

## Build notes
- Parser/serializer is `relationsFile.ts`; keep the on-disk shape a literal
  Markdown table so it stays readable and diff-friendly.
- Relations are their own dimension, independent of boards (epic C) — but they
  share the NotePanel and task typeahead (D02, D03).
- Store the relation *directed* (`From`→`To`); render the inverse view (e.g.
  "blocked by") from the same rows rather than storing both directions.
- Relation types are extensible (D04); `relations.md` itself imposes no fixed
  vocabulary beyond the three-column shape.
