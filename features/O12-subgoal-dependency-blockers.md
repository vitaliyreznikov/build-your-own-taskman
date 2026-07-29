---
id: O12
epic: O — Structured task documents (v2)
title: Subgoal-to-subgoal dependency blockers
size: M
requires: [O03, O06]
novel: true
---

## What
A fourth blocker kind: **a subgoal can be blocked by another subgoal of the same
task**.

```json
{
  "id": "g3",
  "text": "Roll prod",
  "state": "todo",
  "blockers": [
    { "kind": "subgoal", "sg": "g2", "note": "needs the cert issued first" }
  ]
}
```

`sg` is the id of another subgoal **in the same document**. `note` is optional prose
explaining the edge. Nothing else — no `type`, no direction flag, no strength. The
record reads one way only: *this subgoal waits for `sg`*.

**Resolution is one hop.** The blocker is unresolved while the referenced subgoal's
authored `state` is not `done`, and resolved the moment it is. The app does not
follow the target's own dependencies, does not consult the target's blockers, and
does not consider whether the target has unfinished children. `done` on the target
is the whole test, because `done` is the one fact the author controls directly (O03)
and the one an agent can act on without re-deriving a graph.

Consequences of one-hop, all deliberate:

- **Cycles are inert.** `g3` waits on `g5` and `g5` waits on `g3` makes both blocked
  and neither resolvable — visibly wrong on screen, but it cannot hang, recurse, or
  produce a different answer depending on where the walk started. The app detects a
  cycle and marks the chips on it, so the author sees the mistake; it does not
  refuse the document, rewrite it, or drop an edge to break the cycle.
- **A transitive chain still sequences correctly.** `g5 → g3 → g2` works without
  transitive resolution: `g5` unblocks when `g3` is done, and `g3` could only be
  done after `g2`. The chain is enforced by the order things actually get done, not
  by a closure computation.

Three reference cases are named, not left to the reader:

- **Self-reference** (`sg` equals the subgoal's own id) is a typo with no coherent
  meaning. Drop the record on read, same as any blocker failing its type guard
  (O03), and keep the subgoal.
- **A dangling reference** (no subgoal with that id exists) counts as **unresolved**
  and renders as an explicit *unresolved reference* chip naming the missing id. It is
  not silently satisfied. Being wrongly blocked is loud and harmless; being wrongly
  unblocked starts work out of order, which is the failure this feature exists to
  prevent.
- **A cross-task reference** is not this feature. Task-to-task sequencing already
  has a home in `relations.md` (D01). An `sg` value that is not a plain subgoal id
  of this document is a dangling reference and is treated as one.

Presentation follows O03: the chip carries the kind and its human-readable detail —
the target's id, its text, and its current state ("waits for **g2** — get the cert
issued *(doing)*"). The chip is a link: activating it expands and reveals the target
subgoal in the tree, since "what am I waiting for" is unanswerable if the answer is
collapsed three branches away. Everything remains **read-only** (O06): the view
renders edges, it never adds, edits, or removes them.

The dependency changes what the tree *says*, never what order it renders in. Rows
stay in document order (O06) and the open-only filter (O10) is unaffected: a blocked
subgoal is unfinished work and is always kept.

## Why it exists
A v2 task decomposes into subgoals precisely because the work has structure — and
most of that structure is **order**. "Issue the cert, then point DNS at it, then
roll prod" is three subgoals with two hard edges, and until now the document could
express the three but not the two. Nesting is the wrong tool for it: `children[]`
means *part of*, not *after*. Forcing sequence into nesting makes a fake parent
("Phase 1") whose only content is the ordering, and it still cannot say that a step
in one branch waits on a step in another — which is the common case, because
sequence cuts across the decomposition rather than following it.

So the order lived in prose, in `body_state`, or in nobody's head. The cost lands on
whoever picks up the task next, which is increasingly an agent starting a per-subgoal
terminal (O09): given three `todo` subgoals it will pick one, and without the edges
it has no way to know that two of them will fail on contact. A blocker that says
"waits for g2" is the same instruction the human would have given, in the one place
the agent already reads.

The reason it is a **blocker kind** rather than a new `deps` array or a row in
`relations.md` is the invariant from O03: *blockers are the data, blocked is a view
over it.* A dependency is a reason work cannot proceed, which is exactly what a
blocker is; giving it a separate home would create a second place to look for
blocked-ness and a second thing to keep in sync. As a blocker it inherits
everything already built — the derivation, the chip, the roll-up, the index counts,
the forward-compatible round trip — and adds one predicate.

One hop rather than transitive closure is the other deliberate cheapness. A
transitive rule buys nothing a human can observe (the chain sequences itself) and
costs a cycle-safe traversal, a memoisation layer, and an answer that changes with
traversal order the day someone authors a loop. The stated failure mode of a cycle
under one-hop resolution — both ends blocked, visibly — is a *better* outcome than
any automatic repair, because the document is genuinely wrong and only its author
can say which edge was the mistake.

> Design intent, in the author's words: *"a subgoal waiting on another subgoal is
> just another blocker — the point is that nothing in the app has to know it's
> special."*

## Acceptance criteria (EARS)
- The system shall accept a blocker record of kind `subgoal` carrying `sg`, the id of
  another subgoal in the same document, and an optional `note`.
- The system shall treat a `subgoal` blocker as unresolved while the referenced
  subgoal's authored state is not `done`, and as resolved when it is `done`.
- When a `subgoal` blocker is unresolved and the blocked subgoal's state is not
  `done`, the system shall derive the blocked subgoal as blocked, using the same
  derivation as every other blocker kind.
- When the referenced subgoal's state changes to `done`, the system shall stop
  presenting the dependent subgoal as blocked without any edit to the dependent
  subgoal.
- The system shall resolve a `subgoal` blocker one hop only, and shall not consult
  the referenced subgoal's own blockers, dependencies, or children when deciding
  whether the blocker is resolved.
- When a `subgoal` blocker's `sg` names no subgoal in the document, the system shall
  treat the blocker as unresolved and shall render it as an unresolved reference
  naming the missing id.
- When a `subgoal` blocker's `sg` equals the id of the subgoal carrying it, the
  system shall drop that blocker record and keep the subgoal and its other blockers.
- When a `subgoal` blocker record is missing `sg` or `sg` is not a non-empty string,
  the system shall drop that blocker record and keep the subgoal and its other
  blockers.
- While a `subgoal` blocker is rendered, the system shall present the referenced
  subgoal's id, its text, and its current state, together with the `note` when one is
  present.
- When the user activates a rendered `subgoal` blocker, the system shall reveal the
  referenced subgoal in the tree, expanding any collapsed ancestors of it.
- When the dependency edges of a document form a cycle, the system shall render every
  subgoal on that cycle as blocked and shall mark the participating blockers as
  circular, and shall not fail to parse, drop an edge, or alter the document.
- The system shall count a subgoal blocked by a `subgoal` blocker in the same blocked
  roll-ups and index counts as any other blocked subgoal.
- The system shall preserve `subgoal` blocker records, including unknown extra
  fields, across a read-and-write round trip.
- When the open-only filter is on, the system shall render a subgoal blocked by a
  `subgoal` blocker, because it is unfinished work.
- The system shall not add, edit, remove, or reorder dependency edges from the task
  view, and shall not reorder the subgoal tree on account of dependencies.

## Build notes
- Resolution needs sibling context, which `isBlocked(sg)` (O03) does not have. Build
  one `Map<id, state>` per document read in the same depth-capped walk that computes
  the roll-ups, and pass it to the predicate:
  `resolved(b, states) = b.kind === 'subgoal' && states.get(b.sg) === 'done'`.
  Do not thread the whole document into the per-subgoal predicate, and do not look
  the target up by walking the tree per chip — that is quadratic on a deep document.
- The id map is also the dangling-reference test (`!states.has(b.sg)`) and the source
  of the chip's label, so building it once serves all three.
- Cycle marking is a separate, single pass over the edge set — Tarjan or an iterative
  colour walk — done at index time, not per render. Its only output is the set of
  subgoal ids on a cycle; the chip reads that set. Resolution itself never traverses,
  so a cycle cannot recurse even if this pass is skipped.
- Keep the kind string exactly `subgoal` and the field exactly `sg`, matching the
  timeline's citation field (O05). Two names for "the subgoal this refers to" would
  guarantee agents write the wrong one.
- An older app renders this as a neutral unknown-kind chip (O03) and preserves it on
  write — so a document authored with dependencies degrades to "there is a blocker
  here" rather than to a lie. That is the whole reason to extend the blocker union
  instead of adding a top-level key.
- The chip's reveal action reuses the existing collapse state keyed by subgoal id
  (O06): expand each ancestor of the target, then scroll it into view. Do not
  re-render the tree with new keys, which would discard every other branch's
  collapse state.
- Update the agent-facing document contract in the same change as the parser. An
  agent that does not know the kind exists will not write it, and this feature is
  worth nothing unless the thing authoring the subgoals emits the edges.
- Test fixtures worth having: a straight chain of three, a cross-branch edge between
  two different parents, a two-node cycle, a self-reference, a dangling `sg`, and a
  dependency whose target is `done` (must not be blocked).
