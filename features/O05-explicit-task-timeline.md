---
id: O05
epic: O — Structured task documents (v2)
title: Explicit task timeline as JSONL
size: S
requires: [O01]
novel: false
---

## What
Each v2 task may have a **timeline** at `I<N>/timeline.jsonl` — one JSON object per
line, **oldest first, append-only**:

```
{"at":"2026-07-21T09:20:00Z","text":"Staging dual-serve landed; cert ISSUED.","kind":"milestone","sg":"sg-2"}
{"at":"2026-07-22T11:05:00Z","text":"Blocked: waiting on review of PR #8319.","kind":"blocked","sg":"sg-3"}
{"at":"2026-07-23T08:41:00Z","text":"Decided to keep CloudFront in the PCI account rather than move the ALB.","kind":"decision"}
```

Fields: `at` (ISO-8601), `text` (one line, human-readable), optional `kind`,
optional `sg` citing a subgoal id (O03).

The timeline is written **explicitly** — by an agent when it decides an event is
worth recording, or by the human. It is **never** auto-appended per tool call, per
file edit, or per session start/end. `tail` of this file is the latest state.

An entry belongs when, and roughly only when:

- a **phase completes** (a subgoal moved to done, a deploy landed, a PR merged),
- a **decision** is made (an approach chosen, an option rejected, and why),
- a **blocker appears or clears**.

## Why it exists
`body_state` answers "where does this stand?" but it is *overwritten* — it has no
memory of how it got here. The questions the timeline answers are the ones that
otherwise require archaeology: when did this stall, was this decided or did it just
happen, what changed since I last looked.

**The reason it is explicit is that the automatic version is worthless.** A timeline
appended on every tool call, or on every session, records that work happened — which
is the one thing you can already see. It would grow hundreds of entries a week, and
the three entries that matter would be buried among them. The signal *is* the
editorial judgment: someone decided this was worth a line. Cheap to write, so it has
to be expensive enough to mean something. A chatty timeline is noise, and noise gets
skipped, and a skipped timeline may as well not exist.

JSONL for the same reasons as the KB (O04): appends can't corrupt earlier entries, a
grep hit is a whole event, and the git history reads as one line per event. Oldest
first so `tail` is the recent past and `tail -1` is the current state — the cheapest
possible "catch me up" for an agent starting cold.

## Acceptance criteria (EARS)
- The system shall read a v2 task's timeline from `I<N>/timeline.jsonl` as one JSON
  object per line, each with `at` and `text` and optional `kind` and `sg`.
- When `I<N>/timeline.jsonl` does not exist, the system shall treat the timeline as
  empty and shall not report an error.
- When a line fails to parse or fails its type guard, the system shall drop that line
  and load the rest of the file.
- When the system records a timeline entry, it shall append one newline-terminated
  line and shall not modify or rewrite existing lines.
- The system shall not append a timeline entry automatically on tool calls, file
  edits, session start, or session end.
- When an entry cites `sg`, the system shall associate it with that subgoal, and when
  the cited subgoal no longer exists the system shall still render the entry.
- When the system displays the timeline, it shall order entries newest-first for the
  reader while the file remains stored oldest-first.
- When entries carry equal or out-of-order `at` values, the system shall preserve
  file order as the tiebreak rather than dropping or reordering entries.

## Build notes
- Same reader/writer primitives as O04 — build them once, in one JSONL module, and
  use them for both side files.
- Append-only means append-only: no editing past entries in the app. A wrong entry is
  corrected by a new entry. This keeps the file safe for concurrent appends from an
  agent and the app.
- Store oldest-first on disk (append order == chronological order, which is what
  makes `tail` correct) and reverse at render time. Do not store newest-first — that
  turns every append into a full rewrite.
- `kind` is an open vocabulary (`milestone`, `decision`, `blocked`, `unblocked`,
  `note`…). Use it only for an icon or a filter; never require it, and render an
  unknown kind neutrally.
- Keep `text` to one line. Prose that needs paragraphs belongs in `body_state` (or a
  file under `I<N>/`) with the timeline entry pointing at it.
- Put the "when to write an entry" rule in the agent guide (AGENTS.md) verbatim —
  phase complete, decision made, blocker appeared or cleared. Without that line in
  the prompt, agents default to either silence or narration.
- Do not wire the H03 lifecycle hooks or the status markers (H04) into this file.
  Those are the transient live-status channel; conflating them with the durable
  timeline is exactly the auto-append failure this feature rejects.
