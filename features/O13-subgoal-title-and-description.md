---
id: O13
epic: O — Structured task documents (v2)
title: Subgoal title vs description
size: S
requires: [O03, O06]
novel: false
---

## What
A subgoal (O03) gains one optional field, `title`: a short single-line headline.
The existing `text` keeps its meaning unchanged — it is the full description, as
long as it needs to be — and is **not** shortened, split or rewritten by this
feature.

```json
{ "id": "g2_7", "title": "Security review of redaction + log gateway",
  "text": "Scoped to the security-specific pieces only: …", "state": "todo",
  "blockers": [], "children": [] }
```

Everywhere a subgoal is named to a human in one line, the system **prefers
`title` and falls back to `text`**: the tree row (O06), the dependency echo and
blocker tooltip (O12), the PR-blocker rows in the PRs view (O07), and the
subgoal terminal's opening prompt (O09).

In the tree, a subgoal that has both fields renders its title as the row and
keeps the description one click away, collapsed by default, keyed by subgoal id
in the same local view state as the branch collapse (O06/O10). The description is
never *only* reachable through the raw-document view — the row itself must be
able to reveal it, and a row with a description says so before it is clicked.

A subgoal with no `title` renders exactly as it does today: `text` in the row,
nothing collapsed, nothing truncated. There is no migration and no derived
title — an untitled subgoal is a subgoal whose author hasn't titled it yet, not
an error, and the app never invents a headline by cutting prose at a length
limit.

Search (O07) indexes both fields. A `title` that duplicates the `text` verbatim
is treated as no description to reveal.

## Why it exists
`text` is a single free-form string and it absorbed two jobs: what this subgoal
is, and everything known about it. On live documents the second job wins. Measured
across the eight v2 documents in the author's own repo: mean `text` length 400–670
characters on the four big ones, worst case **12 122 characters in one subgoal**
of a 104-subgoal document. Rendered as one span in a flex row, that is a
paragraph-wall that pushes the state marker and blocker chips out of alignment and
makes the tree unreadable as a tree — which is the entire reason the tree exists.
Scanning "what is left" then costs a full read of every node.

The fix is not "write shorter subgoals". The long text is *worth having*: it is the
scope, the decision and the caveats, and an agent picking up a subgoal terminal
needs it. What was missing is a name for the thing, so the two audiences can be
served by the same record — the human scans titles and opens the one that matters,
the agent reads the description.

`title` is optional and additive because the documents are the API: agents edit
them with ordinary file tools, several sessions and git sync write the same file,
and a schema change that *requires* a new field would make every existing document
retroactively wrong. Falling back to `text` keeps every untitled document exactly
as readable as it was, so titles can arrive one document at a time as agents touch
them. Deriving a title from the first N characters was rejected: it silently mangles
prose that was never written to be cut ("Phase 0 · PREREQUISITE — target
architecture is decided (VictoriaMetric…"), and it would hide the fact that a
subgoal still needs a real headline.

Preferring the title in the terminal prompt matters for a second reason: the prompt
that launches a subgoal session interpolates this string, and a 12 000-character
description belongs in the document the agent is about to read, not in the sentence
telling it which subgoal it owns.

## Acceptance criteria (EARS)
- The system shall accept an optional `title` string on every subgoal at any depth,
  and shall preserve it across a read-modify-write of the document.
- When a subgoal has a non-empty `title`, the system shall render that title as the
  subgoal's row in the structured view.
- When a subgoal has no `title`, the system shall render its `text` as the row,
  unchanged and untruncated.
- When a subgoal has both a `title` and a `text` that differs from it, the system
  shall offer a control on the row that reveals the full description in place, and
  shall indicate on the row that a description exists.
- When a task is opened, the system shall render every subgoal's description
  collapsed.
- When the user reveals or hides a description, the system shall keep that state in
  local per-viewer state keyed by subgoal id, and shall not write it into
  `I<N>.json` or any side file.
- When a subgoal's `title` equals its `text`, the system shall not offer a reveal
  control.
- Where a subgoal is named in one line outside the tree — the dependency echo, the
  dependency tooltip, the PR-blocker rows, the subgoal terminal prompt — the system
  shall use `title` when present and `text` otherwise.
- When a document is searched, the system shall match against both `title` and
  `text`.
- When a subgoal's `title` contains newlines or leading/trailing whitespace, the
  system shall normalise it to a single trimmed line rather than breaking the row
  layout.
- When an agent writes a subgoal with only `text`, the system shall treat the
  document as valid and shall not report a missing title as an error.

## Build notes
- One helper next to the O03 predicates — `subgoalLabel(sg) = sg.title || sg.text`
  — and every one-line render site calls it. Do not spread `sg.title ?? sg.text`
  across call sites; the fallback is the contract, and a site that forgets it shows
  a blank row.
- A second helper answers the reveal question: the description is `sg.text` when a
  title exists and `text !== title`, else empty. That keeps "is there anything to
  reveal" out of the component.
- Normalise `title` at parse time in the O01 reader: coerce non-strings away,
  collapse whitespace, trim. The reader already drops malformed records rather than
  throwing — a bad title must degrade to "no title", never to a broken row.
- The subgoal normaliser rebuilds each node field by field, so a new field is
  **dropped unless it is added there** — unlike unknown top-level document keys,
  which survive in `_extra`. Adding `title` to the reader is what makes it
  round-trip.
- Reuse the existing per-id local view state for the reveal, and give it its own
  map: folding a description must not collapse the branch's children, and expanding
  a branch must not dump every description on screen.
- The row's reveal control should be the title itself where possible rather than
  new chrome — the tree row is already crowded with state marker, id, blocker chips
  and terminal launchers.
- Keep the description block outside the flex row, as its own wrapping block, so a
  long paragraph can never re-flow the chips.
- Document the field in the agent-facing schema notes alongside `text`, with the
  length guidance (one line, ~80 characters, no markdown) — the field is only
  useful if the agents writing these documents fill it in.
- Fixtures worth having: a subgoal with title + long text (row is the title,
  description reveals), title only (no reveal control, row is the title), text only
  (renders as before), title identical to text (no reveal control), and a title
  containing a newline (renders on one line).
