---
id: O07
epic: O — Structured task documents (v2)
title: Document index cache + blockers discovered from documents
size: M
requires: [O01, O03, O04, L01]
novel: false
---

## What
Two things that fall out of having structured documents on disk.

**1. A derived index over every document.** An **mtime-keyed** index over all
`I*.json` files (and their `I<N>/kb.jsonl` / `timeline.jsonl` side files) holding
only the small derived facts the app needs to render boards, badges and search:

- title, subgoal counts (`total` / `done` / `blocked`),
- the list of unresolved blockers, with kinds,
- every `{kind:"pr"}` url found in the document,
- flattened search text (bodies + subgoal text + KB `q`+`a`).

It is invalidated **per file by the watcher** (B11), **rescanned periodically** as a
safety net, and **persisted to a machine-local, gitignored cache** treated as purely
advisory: any file whose mtime or size disagrees with the cached entry is reparsed,
and a corrupt or unreadable cache means "empty index", never "no documents".

**2. PR blockers discovered from documents.** The PR-review poller (L01) stops
reading only `relations.md`. Its URL set becomes the **union** of

- `blocked-by-pr` rows in `relations.md`, and
- every `{kind:"pr", url}` blocker on every subgoal of every v2 document,

so a **subgoal-level** PR blocker gets the same live review state, the same
notification, and the same auto-resume of the task's agent as a task-level one.
Excluded from the set: documents belonging to closed/done tasks, and blockers on
subgoals that are `done` — otherwise the app polls a merged PR forever.

The same index answers **cross-task KB search**.

## Why it exists
Once the app needs to show "3 of 7 subgoals, 1 blocked" on a card, a naïve
implementation parses every document on every render. With a few hundred tasks
that's thousands of parses per board paint. The index exists so nothing parses a
document on demand.

**It is a derived cache, never a source of truth — and that is why there is no
database.** A database would give indexes, transactions and queries, and would take
the property the whole system rests on: the files are canonical, greppable,
git-diffable, and readable without the app. The moment a DB holds anything the files
don't, an agent editing a file by hand (O01: files are the API) is editing a stale
copy, and every hand edit needs a reconciliation story. Keeping the index strictly
derived means the correctness test is trivial: **deleting the cache must be safe** —
worst case the app is slow for one scan. Nothing is ever recoverable *only* from the
index; if the answer isn't in a file, it doesn't exist.

Discovering PR blockers from documents is the same idea applied to the poller: the
blocker is already written down where the work is, so the poller should read it there
instead of asking the human to also register it in a second table. Subgoal-level PR
blockers are the common case in practice — a task rarely waits on a PR as a whole, a
step does.

## Acceptance criteria (EARS)
- The system shall maintain an index over every `I*.json` document keyed by file
  mtime, holding title, subgoal counts, unresolved blockers, PR urls, and flattened
  search text.
- When the watcher reports a change to a document or one of its side files, the system
  shall invalidate and reparse only that task's entry.
- The system shall rescan all documents on a periodic interval as a safety net for
  changes the watcher missed.
- When the system starts, it shall load the persisted cache and shall reparse any file
  whose mtime or size differs from the cached entry.
- If the persisted cache is missing, unreadable, or corrupt, then the system shall
  start from an empty index and rebuild it, and shall not report that there are no
  documents.
- When parsing a document fails, the system shall retry once and, if it fails again,
  shall keep the last-good derived facts for that task rather than dropping its
  blockers.
- The PR-review poller shall poll the union of `blocked-by-pr` relation rows and every
  `pr` subgoal blocker found in the index.
- When a PR discovered from a subgoal blocker receives its first human review, the
  system shall react exactly as for a relation-table PR blocker (notify if a terminal
  is live, otherwise open the task's terminal with the review context).
- The system shall exclude from the poll set PR blockers on subgoals whose state is
  `done` and documents whose task is in a closed or done column.
- When the user searches across tasks, the system shall answer from the index over
  flattened text, including v2 bodies, subgoal text, and KB questions and answers.
- The system shall be able to rebuild the entire index from the files alone, and
  deleting the persisted cache shall not lose any data.
- The persisted cache shall be machine-local and gitignored.

## Build notes
- Entry shape: `{ mtime, size, title, counts, blockers[], prUrls[], text }` per task
  id, plus a per-side-file `{ mtime, size }` so a KB append invalidates only the
  search text. Keep entries small — this is a summary, not a copy of the document.
- **Agent writes are not rename-atomic.** The app writes through A06; an agent using
  ordinary file tools does not. So a read that catches a document mid-write is normal,
  not exceptional. Retry once after a short delay (~150ms); on a second failure keep
  the previous entry and mark it stale. Dropping to "no blockers" on a transient
  parse failure is the bug to avoid — it would silently un-block a task and stop its
  PR polling.
- Store `prUrls` with their owning subgoal id so the poller's reaction can name the
  subgoal in the notification and the resume prompt, and so review state can be
  attached back to the right chip in O06.
- Reuse L01's `.pr-status.json` for review state and the fired-once baseline. Nothing
  about the *reaction* changes; only the URL discovery does. Dedupe the union by
  normalized URL so a PR blocking both a task row and a subgoal fires once.
- Poll-set exclusions belong in the query, not the reaction: filter by column
  (A04/`tasks.md`) and subgoal state when building the set each tick. Otherwise the
  set only grows and the API cost grows with the archive.
- Search must index **flattened text**, not raw JSON. Concatenate `body_human`,
  `body_state`, subgoal `text`, KB `q` and `a` into one lowercase string per task;
  matching against raw JSON returns hits on key names, escapes and timestamps.
- Periodic full rescan on a long interval (minutes) and cheap: `stat` first, parse only
  on mtime/size mismatch. On macOS the watcher misses events under load and across
  some editor save patterns, which is the whole reason the safety-net scan exists.
- Persist the cache next to the other machine-local files (`.llm-status.json`,
  `.pr-status.json`) and gitignore it. Version the cache file with a schema number and
  discard it wholesale on mismatch — cheaper and safer than migrating an advisory
  cache.
- Do not let any feature read a v2 fact from the index that it cannot also get from
  the file. The index is a performance layer; if a code path can only be satisfied by
  the cache, the design has drifted into a database.
