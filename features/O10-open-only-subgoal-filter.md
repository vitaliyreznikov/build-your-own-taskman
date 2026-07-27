---
id: O10
epic: O — Structured task documents (v2)
title: Open-only subgoal filter
size: S
requires: [O03, O06]
novel: false
---

## What
The structured view (O06) gains one control above the subgoal tree: **show open
only**. With it on, the tree renders only the work that is still live; finished
branches disappear.

The rule that defines the feature is the one about ancestors. A subgoal is kept if

```
keep(sg) = sg.state !== 'done' || any descendant of sg has state !== 'done'
```

So a `done` parent that still contains unfinished children **stays visible**.
Hiding it would take its open children with it, which is the single outcome this
filter must never produce. Such a parent is present as *structural context*, not as
active work, and is rendered de-emphasised (dimmed, no emphasis on its state
marker) so the tree does not claim work is in progress where it is not.

"Done" means the authored `state` (O03). Blocked-ness is irrelevant: a blocked
subgoal is unfinished work — usually the most interesting unfinished work on the
screen — and is always kept.

The control names what it is switching between, e.g. **"show open only (39 open of
62)"**, so the user knows the size of what is about to be hidden before hiding it.
It defaults to **off**.

The toggle is **per-viewer UI state**. It is persisted locally, in the same place
as the other reading preferences (panel width, board sort mode), and is **never
written into the document** — it is a preference of the person reading, not a fact
about the task. Consequently it is presentational only: the subgoal tally on the
task card, the roll-ups in the index cache (O07), and the blocker chips that feed
the PR poller (L01/O07) are computed from the document and are unaffected by it.
Per-branch collapse/expand state is likewise independent — toggling the filter does
not collapse, expand, or forget anything.

## Why it exists
A structured task accumulates completed subgoals faster than it sheds them. Nothing
removes a `done` node — it is the record of what happened, and the timeline (O05)
cites it — so on a task that has been running for weeks the tree is mostly history:
62 subgoals of which 23 are done, and the answer to "what is left" is somewhere
between them. But "what is left" is the entire value of the tree. Finished branches
do not merely take up space, they actively obscure that answer, because reading the
tree means visually filtering them out by hand on every open.

The filter is the cheap version of that work: one predicate, done once, correctly.
The reason it is a spec and not a one-line `filter()` is the ancestor case. The
naive predicate — drop every `done` node — silently deletes open leaves whose parent
happens to be marked done, which is common (a parent closed slightly early, or
closed because its main line landed while a follow-up child remains). A filter whose
failure mode is *hiding unfinished work* is worse than no filter at all, so
retention of done ancestors is a hard requirement, and the de-emphasis exists so
that retaining them does not overstate them.

It defaults to off because a reader should never be shown a partial tree they did
not ask for, and it lives in local view state because writing a reading preference
into a git-synced document turns a UI click into a commit and a conflict (same rule
as collapse state in O06).

## Acceptance criteria (EARS)
- While a v2 task's structured view is open, the system shall offer an "open only"
  toggle above the subgoal tree.
- When a task is opened for the first time on this machine, the system shall default
  the toggle to off and render the complete subgoal tree.
- When the toggle is on, the system shall render a subgoal if its state is not `done`
  or if any of its descendants has a state other than `done`, and shall hide it
  otherwise.
- When the toggle is on and a `done` subgoal is retained because a descendant is
  unfinished, the system shall render that subgoal de-emphasised rather than as
  active work, and shall render its unfinished descendants normally.
- When the toggle is on and a subgoal is unfinished, the system shall render it
  regardless of whether it is blocked, so that a blocked subgoal is never hidden.
- When the user changes the toggle, the system shall persist the new value in local
  view state and shall not write it into `I<N>.json` or any side file.
- When the user reopens the app, the system shall restore the toggle's last value
  from local view state.
- While the toggle is rendered, the system shall state the count of open subgoals and
  the total count of subgoals for the task being viewed.
- When the toggle changes, the system shall leave the task card's subgoal tally, the
  document index roll-ups, and the set of blockers polled for review state unchanged.
- When the toggle changes, the system shall preserve each branch's collapsed or
  expanded state.
- When the toggle is on and every subgoal in the task is `done`, the system shall
  show an explicit empty state naming the filter as the reason rather than an empty
  tree.

## Build notes
- The predicate is a pure function next to `isBlocked` in the O01 document module:
  `hasOpenWork(sg) = sg.state !== 'done' || sg.children.some(hasOpenWork)`. Compute
  it in one post-order walk over the already-parsed tree and memoise per node — do
  not re-walk the subtree per rendered row, which is quadratic on a deep tree.
- The walk reuses O01's depth cap. It runs on the parsed document, so it costs
  nothing on documents that are already in the panel's render path.
- Keep filtering in the renderer only. Nothing in the index cache (O07), the poller
  (L01), or the card summary reads the toggle; if any of them ever needs a filtered
  count, give it the predicate, not the UI flag.
- The "open" count in the label is `count(sg where sg.state !== 'done')` over the
  whole tree — the plain open count, not the number of *rendered* rows, which
  includes retained done ancestors and would be a confusing number to show.
- De-emphasis is a visual state, not a third state marker: keep the authored `done`
  marker, lower contrast on the row. Do not invent a "contains open work" badge —
  the visible children are the badge.
- Store the flag under one key in the same local view-state store as panel width and
  board sort mode. It is per viewer, not per task: a per-task key multiplies
  invisible state without answering a question anyone asked.
- Collapse state is keyed by subgoal id (O06), so it survives filtering for free as
  long as the filter removes rows rather than rebuilding the tree with new keys.
- Test fixtures worth having: a done parent with one `todo` leaf (must render both),
  a done parent with only done children (must hide the whole branch), a `todo`
  subgoal with an unresolved `answer` blocker (must render), and a fully done
  document (must show the empty state).
