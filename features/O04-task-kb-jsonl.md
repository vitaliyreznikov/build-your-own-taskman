---
id: O04
epic: O — Structured task documents (v2)
title: Task KB as JSONL
size: S
requires: [O01]
novel: false
---

## What
Each v2 task may have a **knowledge base** at `I<N>/kb.jsonl` — one JSON object per
line, no enclosing array:

```
{"id":"kb-1","q":"Which account holds the tokenizer CloudFront distribution?","a":"PCI account 394364755619 — not the main prod account.","at":"2026-07-21T09:14:00Z"}
{"id":"kb-2","q":"How do I apply a hail datafix?","a":"Jenkins job database/apply_datafix with MIGRATION_VERSION=<numeric> and ARE_YOU_SURE=true. Auto-apply is broken.","at":"2026-07-22T16:02:00Z"}
```

Fields: `id` (stable), `q` (the question), `a` (the answer), `at` (when learned).
The KB holds **reusable question→answer facts learned while working the task** —
the things you had to find out and would otherwise find out again. There is no
`tags` field.

Cross-task search is `grep I*/kb.jsonl`.

## Why it exists
Most of the time an agent spends on a mature task is re-derivation: which account,
which job, which flag, why the obvious approach doesn't work. Those answers were
already found — they just landed in a chat transcript that is gone, or in a wall of
prose nobody greps. A per-task KB makes them a first-class, addressable artifact.

**JSONL rather than a JSON array is deliberate, and buys three things:**

1. **One grep hit returns one complete record.** `grep 394364755619
   I*/kb.jsonl` prints the whole fact — question, answer, timestamp — with no
   parsing and no context window. In a pretty-printed array a hit returns a
   fragment of a line and you have to reconstruct which object it belonged to.
2. **Appending cannot corrupt the rest of the document.** A new fact is
   `>> kb.jsonl` — no read, no re-serialize, no closing bracket to get right.
   A truncated write costs the last line; an array with a bad tail is unparseable
   in full. This matters exactly because untrusted writers append (O01: files are
   the API).
3. **Git diffs are one line per fact.** The history of what the task learned reads
   as a list of additions, not as a reindented blob.

Two access patterns fall out for free, and both are wanted:

- **Table of contents** — `jq -r .q I<N>/kb.jsonl` lists every question in a few
  tokens. Cheap enough for an agent to scan on every session start.
- **Full-text search over question AND answer** — plain `grep` on the file. This is
  the **default** search, because the discriminating token is usually in the
  *answer*: an account number, a job name, an error string. A question-only search
  would miss most real lookups.

**No tags, on purpose.** Grep over the full record already does what tags were for,
tags require agreeing on a vocabulary that no two agents will agree on, and an
untagged-because-forgotten fact becomes invisible. Instead, the discipline is on
phrasing: **write questions with their keywords in them** ("Which Jenkins job
applies a hail datafix?" not "About datafixes"), so the TOC scan is genuinely
useful and the question line is itself searchable.

## Acceptance criteria (EARS)
- The system shall read a v2 task's KB from `I<N>/kb.jsonl` as one JSON object per
  line, where each object has `id`, `q`, `a`, and `at`.
- When `I<N>/kb.jsonl` does not exist, the system shall treat the task's KB as empty
  and shall not report an error.
- When a line fails to parse or fails its type guard, the system shall drop that
  line and load every other line in the file.
- When a trailing line is incomplete (an interrupted append), the system shall
  ignore it and load the preceding lines.
- When the system adds a KB entry, it shall append a single line terminated by a
  newline and shall not rewrite the existing lines.
- When the system writes a KB entry, it shall emit it as one physical line with no
  embedded raw newlines.
- The system shall support listing a task's KB questions without loading the
  answers.
- The system shall support searching a task's KB over both the question and the
  answer text, and shall use that combined search as the default.
- The system shall not require a tags field, and shall preserve unknown fields on an
  entry it rewrites.

## Build notes
- Reader: split on `\n`, skip blank lines, `JSON.parse` each line in `try/catch`,
  type-guard, collect. Never let one bad line reject the file — that is the whole
  point of the format.
- Writer: append-only (`appendFile`) with a leading-newline check so a file whose
  last line lacks its terminator doesn't get two records glued together. Full
  rewrites (edit, delete) go through the atomic writer (A06); appends do not need
  it and should not take it, or you lose the append's crash-safety.
- Escape newlines inside `a` (JSON string escaping already does this — just don't
  pretty-print). One record must be exactly one line, or every grep and `jq -r`
  guarantee above breaks.
- Ids are stable (`kb-<n>`, max + 1) because timeline entries and prose may cite
  them. Deleting an entry leaves a gap; don't compact.
- Search inside the app (B09, O07) must run over **flattened text** — the
  concatenation of `q` and `a` — not over the raw JSON line, or the user matches
  field names, escape sequences and timestamps and gets noise.
- Cross-task search is literally `grep I*/kb.jsonl` from the data directory; the
  in-app equivalent is the derived index (O07). Keep both, and keep them agreeing on
  what "match" means.
- Put the phrasing rule in the agent guide (AGENTS.md), not only here: keywords in
  the question, one fact per entry, answer complete enough to act on without opening
  the task.
