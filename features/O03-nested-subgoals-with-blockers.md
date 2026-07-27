---
id: O03
epic: O — Structured task documents (v2)
title: Nested subgoals with blockers
size: M
requires: [O01]
novel: true
---

## What
`subgoals[]` in a v2 document is a **tree**. Each node is:

```json
{
  "id": "sg-3",
  "text": "Get the cert issued for the new domain",
  "state": "doing",
  "blockers": [ { "kind": "pr", "url": "https://github.com/org/repo/pull/8319" } ],
  "children": []
}
```

`state` is authored as one of `todo | doing | done`. **There is no authored
`blocked` state.** A subgoal is *blocked* iff it has at least one unresolved
blocker and is not `done` — the app derives it on read.

`blockers[]` holds records of three known kinds:

- **`{ kind: "pr", url }`** — waiting on review of a pull request. The app polls it
  and shows live review state (O07); it clears itself when the review lands.
- **`{ kind: "answer", who, asked, since }`** — waiting on a person to answer a
  question. `who` is the person, `asked` is the question, `since` is when it was
  asked. Only a human clears it.
- **`{ kind: "user-action", who, what, since }`** — the owner must do something
  themselves that the agent cannot: run a command on their machine, obtain
  credentials, make a phone call, approve something in a console.

Unknown kinds are **rendered as a neutral chip**, not dropped. Subgoal ids are
stable and are never renumbered.

## Why it exists
A flat checklist tells you what is unfinished but not *why* it is unfinished, and
"why" is the only thing that decides what the human does next. Splitting the wait
into kinds makes the queue actionable: PR blockers resolve themselves, `answer`
blockers are someone else's turn, and `user-action` blockers are the human's own
to-do list — the one pile that never moves unless the human sees it.

**Blocked is derived, not stored.** Adding `"blocked"` to the state enum reads as
the natural fourth option and is a trap for two reasons:

1. *It would be a second source of truth.* The blocker list already says whether
   work can proceed. A stored flag beside it drifts the first time an agent clears
   a blocker and forgets the flag, or sets the flag while removing the blocker — and
   then the app has two answers and no way to pick.
2. *A blocked flag with no blocker record is unactionable.* "Blocked" alone tells
   the human to go read the task and reconstruct the reason. Forcing the blocker to
   exist means the reason is always attached to the claim.

So the invariant is one-directional: blockers are the data, blocked is a view over
it. Clearing the last blocker un-blocks the subgoal automatically, with nothing else
to remember.

Nesting exists because real tasks decompose more than one level (phase → cluster →
step) and a blocker at a leaf must be visible from the root without flattening away
the structure that explains it.

> Design intent, in the author's words: *"blocked isn't a state you type — it's what
> it means to have a blocker you haven't cleared."*

## Acceptance criteria (EARS)
- The system shall accept `subgoals[]` as a tree in which every node has `id`,
  `text`, `state`, and optional `blockers[]` and `children[]`.
- The system shall accept only `todo`, `doing`, `done` as authored `state` values,
  and shall treat an unrecognized value as `todo`.
- The system shall derive a subgoal as blocked when it has one or more unresolved
  blockers and its state is not `done`, and shall not read or write a stored
  `blocked` state.
- When a subgoal's last unresolved blocker is removed, the system shall stop
  presenting that subgoal as blocked without any other edit to the subgoal.
- When a subgoal's state is `done`, the system shall not present it as blocked even
  if blocker records remain on it.
- When a blocker's `kind` is not one of the known kinds, the system shall render it
  as a neutral chip carrying its available fields rather than dropping it.
- When a blocker record fails its type guard (missing `url` on a `pr`, missing
  `who`/`what`), the system shall drop that blocker record and keep the subgoal and
  its other blockers.
- The system shall never renumber or reassign an existing subgoal `id`, and shall
  allocate a new unused id for a new subgoal.
- When the system rolls up a subtree, it shall report a subgoal's blocked or
  unresolved descendants at every ancestor level up to the root.
- While a subgoal is blocked, the system shall present the blocker's kind and its
  human-readable detail (PR url and review state, the question and who owes it, the
  action the owner must take).

## Build notes
- Derive with a pure function: `isBlocked(sg) = sg.state !== 'done' &&
  unresolved(sg.blockers).length > 0`. Keep it in the O01 document module so the
  panel (O06) and the index cache (O07) cannot disagree.
- "Unresolved" for a `pr` blocker is a *live* fact, not a stored one: the poller's
  review state (L01/O07) decides. Treat unknown/unpolled as unresolved so a task
  never looks free because a poll hasn't run yet.
- `answer` and `user-action` blockers have no automatic resolution path. Removing
  the record is the only way to clear them, and only a human (or an agent the human
  told) does it. Do not let a poller, a heuristic, or an age threshold expire them.
- Keep `since` / `asked` as ISO-8601 strings and show elapsed time in the UI — "9
  days waiting on an answer" is the signal, and it is what makes a forgotten
  `answer` blocker visible.
- Subgoal ids are cited by timeline entries (`sg`, O05) and may be cited from the
  KB or a commit message. Renumbering silently invalidates those references, which
  is why ids are allocated (max-suffix + 1, or a short random suffix) and never
  compacted — gaps after deletion are fine.
- Roll-ups: compute counts (`total`, `done`, `blocked`) once per document read via a
  single depth-capped walk (O01 caps recursion), and cache them in the index (O07)
  rather than recomputing per card render.
- Unknown-kind chips are the forward-compatibility hatch: a future blocker kind
  written by a newer agent prompt must survive a round trip through an older app.
  Preserve unknown blocker fields on write, same rule as unknown top-level keys.
