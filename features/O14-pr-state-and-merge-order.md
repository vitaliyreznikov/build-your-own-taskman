---
id: O14
epic: O — Structured task documents (v2)
title: PR blocker carries live state and merge order
size: M
requires: [O03, O06, O07, L01]
novel: true
---

## What
The `pr` blocker (O03) grows two independent halves: **what the PR is doing on
GitHub**, kept current by the poller, and **what has to happen before it may
merge**, authored by whoever writes the document.

```json
{
  "kind": "pr",
  "url": "https://github.com/acme/devops/pull/8285",
  "after": [
    "https://github.com/acme/kubernetes-deployments/pull/52527",
    "https://github.com/acme/gitops/pull/157",
    "the wallet change freeze lifts (2026-08-10)"
  ],
  "state": "approved",
  "since": "2026-07-25 16:04",
  "note": "approved; merge only after the Kyverno guard is live on both clusters"
}
```

**`state` is GitHub's answer and nothing else** — one collapsed enum, first match
wins, derived on every poll:

| value | when |
|---|---|
| `merged` | the PR is merged |
| `closed` | closed without merging |
| `draft` | open, marked draft |
| `changes-requested` | the review decision is changes-requested |
| `approved` | the review decision is approved |
| `in-review` | reviewers assigned or a human review submitted, no decision yet |
| `open` | none of the above |

`since` is the local `YYYY-MM-DD HH:mm` at which the state last **changed** — not
when it was last polled. That distinction is the whole reason it is safe to keep
this in a git-synced file: a poll that finds nothing new writes nothing, so the
document's history reads as the PR's history rather than as a heartbeat.

**`after` is the merge order**, a list of things that must happen before this PR
may merge. An entry is either a **PR url**, which the app resolves itself (met
once that PR is `merged` or `closed`), or **free text**, which the app cannot
resolve and which is therefore met only when a human deletes the line. Both kinds
of entry count identically while present. This is the one place in the document
where a satisfied record is deleted rather than kept — the opposite of the O12
convention — because there is nothing to resolve free text against, and a list
whose entries never leave would stop meaning "what is still in the way".

**`gated` is derived, never stored.** A blocker is gated when its `state` is one
a PR could merge from (`open`, `in-review`, `approved`) and at least one `after`
entry is unmet. Keeping it out of `state` is deliberate: gating has causes the
poller cannot see, so a stored `gated` would be a value only an agent could write
and only an agent could clear — stale by construction. Derived, it is correct the
instant the last gate merges, with no write at all.

The two halves have **different owners, and the document says so**: agents author
`url`, `after` and `note`; the app owns `state` and `since` and overwrites any
hand-edit on the next poll. This is the first thing the app writes into a v2
document that a human did not ask it to write, so it writes surgically — re-read
the file, set those two fields on that one blocker, write back — and only when the
value actually changed.

Everything downstream reads the derived status rather than the raw state:

- The blocker chip says what is true and what is missing:
  `PR #8285 · approved · gated (2)`, its detail naming each unmet entry.
- **The auto-resume reaction (L01) fires on `approved` and not gated** — the edge
  where the PR became actionable — instead of on "a human reviewed it". A review
  that lands on a gated PR is remembered, not announced, and announces itself when
  the last gate clears.
- A `merged` or `closed` blocker resolves itself, exactly as an O12 dependency
  does when its target is `done`. The record stays in the document as history.
- A transition appends one line to the task's timeline (O05), so the document
  gains an audit trail of when each PR moved without anyone writing it down.

## Why it exists
A `pr` blocker used to have one axis: *has a human reviewed it*. Real PRs are
sequenced. A PR can be approved for days and still be unmergeable because two
other PRs in two other repos have to land first — and in that state the app did
the worst possible thing: it rendered the blocker green, said "reviewed", and
opened a terminal telling an agent to go and act on it. The nudge was not merely
noise, it was wrong, and it re-fired on relaunch. Meanwhile the actual ordering
lived in prose in `body_state`, where the next session had to re-read a wall of
text to discover it, and often re-derived it from GitHub instead.

Both halves are needed, and neither substitutes for the other. Without `after`
the app cannot tell an actionable approval from a parked one. Without `state` the
document cannot answer "where is this PR" to anyone who is not the running app —
an agent reading the file, a second machine, `git log`. The status cache the
poller already keeps is machine-local and per-url; it is the right home for
"checked at", the wrong home for the fact the task's own history is made of.

The reason both live **on the existing blocker** rather than in a new structure is
the O03 invariant: blockers are the data, blocked is a view over it. Merge order
is a reason work cannot proceed, which is what a blocker is. Adding a second home
for it would mean two places to look for why something is stuck.

The reason `after` accepts free text is that gating is not a GitHub concept. A
change freeze, a datafix that has to be applied by hand, a customer call on
Thursday — these gate a merge exactly as another PR does, and forcing them into
prose would put half the answer in `note` and half in `after`, which is precisely
the split this feature exists to close.

> Design intent, in the author's words: *"approved but not mergeable is not the
> same as approved, and the app should never again tell me to merge something
> that cannot merge."*

## Acceptance criteria (EARS)
- The system shall accept a `pr` blocker carrying an optional `after` array whose
  entries are PR urls or free text, and shall preserve unrecognised entries verbatim.
- The system shall derive a `pr` blocker's `state` from the pull request on every
  poll, as exactly one of `draft`, `open`, `in-review`, `changes-requested`,
  `approved`, `merged`, `closed`.
- When a `pr` blocker's derived state differs from the state recorded in the
  document, the system shall write the new state and the time of the change into
  that blocker, and shall leave every other part of the document unchanged.
- When a `pr` blocker's derived state equals the recorded state, the system shall
  not write to the document.
- The system shall re-read the document immediately before writing a state change,
  and shall not write when the blocker it is updating is no longer present.
- The system shall treat an `after` entry that is a PR url as met when that pull
  request is merged or closed, and as unmet otherwise.
- The system shall treat an `after` entry that is not a PR url as unmet for as long
  as it is present.
- The system shall poll every PR named in an `after` entry, and shall not fire a
  review reaction for a PR that is only named there.
- While a `pr` blocker's state is `open`, `in-review` or `approved` and at least one
  `after` entry is unmet, the system shall present that blocker as gated and shall
  state how many entries are unmet.
- While a blocker is presented as gated, the system shall name each unmet `after`
  entry and shall present the blocker's `note` alongside them.
- When a `pr` blocker becomes `approved` with no unmet `after` entry, the system
  shall fire the review reaction for the blocked unit of work exactly once.
- While a `pr` blocker is gated, the system shall not fire the review reaction, and
  shall fire it when the last unmet entry becomes met without requiring a new review.
- The system shall treat a `pr` blocker whose state is `merged` or `closed` as
  resolved, and shall keep the blocker record in the document.
- When a `pr` blocker's state changes, the system shall append one entry to the
  task's timeline naming the pull request, the new state and the subgoal.
- The system shall not append a timeline entry for a poll that changes nothing.
- The system shall present a hand-authored `state` or `since` as the current value
  until the first successful poll, and shall replace it thereafter.
- The system shall preserve `after`, `note` and any unknown fields of a `pr` blocker
  across a read-and-write round trip.
- The system shall order the PR list so that blockers that are actionable appear
  above blockers that are gated.

## Build notes
- Ask GitHub for `isDraft` and `reviewDecision` in the same query that already
  fetches reviews. `reviewDecision` is the authoritative approve/changes-requested
  signal; the review list stays for reviewer names and timestamps, which the
  collapsed state deliberately drops.
- The poller's tracked-url set becomes the union of blocker urls and every url
  appearing in an `after`. Track a gate url with an empty task list so the reaction
  loop skips it naturally instead of needing a flag.
- Keep the existing "high-water mark" that stops a review re-firing, but advance it
  only when the reaction actually fires. Advancing it while gated is the bug that
  would silently eat the announcement when the gates clear.
- The document write is a read-modify-write on raw parsed JSON, not on the
  normalised document: normalising and re-serialising would quietly rewrite parts of
  the file the app does not own. Walk the raw subgoal tree, match on subgoal id plus
  url, and write only if something changed.
- Do not put "checked at" in the document. The local status cache already has it,
  and a timestamp that moves every five minutes would make the file a git-noise
  generator and every diff meaningless.
- Derive gated in shared code used by both the task view and the PR list. Two
  implementations of "is this actually actionable" will disagree, and the whole
  point of the feature is that one answer is trustworthy.
- An older app renders the extra fields as an ordinary `pr` chip and round-trips
  them untouched, so a document authored with merge order degrades to today's
  behaviour rather than to a lie.
- Update the agent-facing contract in the same change: gated is the state an agent
  meets most often, and it must know to read `after` and `note` together, and that a
  satisfied free-text entry is deleted by hand.
- Fixtures worth having: an approved PR with two unmerged gates, the same after both
  gates merge (must fire once), a gate that is free text (must never self-clear), a
  draft PR, a merged blocker (must resolve, must stay in the file), and a poll that
  changes nothing (must not write).
