---
id: O07
epic: O — Structured task documents (v2)
title: Document index cache + blockers discovered from documents
size: M
requires: [O01, O03, O04, L01, O09]
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
notification, and the same auto-resume as a task-level one.
Excluded from the set: documents belonging to closed/done tasks, and blockers on
subgoals that are `done` — otherwise the app polls a merged PR forever.

That union is deduplicated on `(task, url)`, and **the document row must win the
tie.** The two rows are not interchangeable: only the document row carries the
subgoal, the merge order and the recorded state (O14). If the `relations.md` row
wins — which is what a naive relations-first concatenation does, and what shipped
until 2026-08-04 — a subgoal blocker that someone also hand-added to
`relations.md` is silently demoted to task-level: the notify-vs-open test below
goes task-wide again, and the O14 write-back is skipped because there is no
subgoal to patch. The symptom is a PR approval that produces a notification (or
nothing) while the blocked subgoal never gets its terminal, and a document whose
`state:` stays stale forever. Both the poller's tracked set and the PRs view's
`prBlocks` snapshot dedupe this way.

The *reaction* differs in one way, and it matters: a blocker that names a subgoal
resumes **that subgoal's terminal** (O09) — `task-<id>-sg-<subgoal>`, carrying the
work-on-subgoal brief — not the task-level terminal with the whole-task brief. The
PR blocks one step, so the agent that wakes up should be scoped to that step.
Naming the subgoal in prose inside the extra prompt is not enough: the surrounding
brief still tells the agent to work the entire task, and on a task with dozens of
subgoals that is the wrong instruction.

**And the notify-vs-open test has to be scoped the same way.** L01 asks "does this
*task* have a live terminal?"; for a subgoal blocker the honest question is "does
*this subgoal* have one?" A big v2 task normally has several agents running at once
— one per subgoal, which is the whole point of O09 — so the task-wide test is true
almost always, and every subgoal review silently degrades to a notification while
the step that was actually unblocked never gets an agent. A live sibling is not a
reason to skip opening: the reviewed PR blocks *this* step, and no agent working
another step will pick it up. Only a live terminal **on that same subgoal** means
someone is already there, and that is the one case that should notify instead —
naming the subgoal, since on a many-subgoal task "PR reviewed — I244" doesn't say
where to look.

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
- The live-terminal test that decides notify-vs-open shall be scoped to the **blocked
  unit of work**: that subgoal for a subgoal blocker, the whole task for a task-level
  (`relations.md`) blocker.
- When a PR discovered from a **subgoal** blocker receives its first human review and
  **that subgoal's own** terminal is live, the system shall fire the same notification
  as for a relation-table blocker — naming the subgoal — and shall not open a terminal.
- When a PR discovered from a **subgoal** blocker receives its first human review and
  that subgoal has no live terminal, the system shall open **that subgoal's** terminal
  (O09, `task-<id>-sg-<subgoal>`, work-on-subgoal brief) rather than the task-level
  terminal — **even when the task-level terminal or other subgoals' terminals of the
  same task are live**.
- Whenever the system opens a terminal in reaction to a review, the extra prompt
  naming the reviewed PR shall reach the agent's first submission — including when a
  tab record for that terminal already exists in the renderer from a previous
  session or a restore.
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
- Reuse L01's `.pr-status.json` for review state and the fired-once baseline. Dedupe
  the union by normalized URL so a PR blocking both a task row and a subgoal fires
  once.
- **Scope the live-terminal predicate, don't add a second one.** L01's predicate takes
  a task id and matches any `task-<id>*` tmux session; give it an *optional* subgoal id
  and, when present, compare the subgoal slug parsed out of the session name (O09) as
  well. Optional, not a new function, because the same predicate also gates K01 autorun
  — which is task-level and must keep matching any of the task's sessions, subgoal ones
  included, or autorun starts a second whole-task agent next to a running one.
- **Thread the subgoal id, not just its text, all the way to the pty.** L01's
  reaction hands the renderer `(taskId, prompt)`; that signature is what forces a
  document blocker into a task-level terminal, because the id it needs to build the
  O09 session name has already been discarded — the poller only kept the subgoal
  *text*, for the prose. Carry `{subgoalId, subgoalText}` on the IPC payload and
  branch to O09's `openSubgoalTerminal`.
- **The renderer's "tab already exists" shortcut is where the review context dies.**
  The task-terminal opener reuses a tab record by name and, in doing so, keeps the
  old record — dropping the `extraPrompt` and `autoClaude` on the new request. A
  restored-but-dead tab therefore starts `claude` with the plain work-on-task prompt
  and no mention of the review at all, which reads as "the reaction fired but the
  agent ignored it". Reusing the tab is right; keeping its stale fields is not — patch
  the pending prompt onto the existing record.
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
