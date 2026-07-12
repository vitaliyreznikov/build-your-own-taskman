---
id: L01
epic: L — PR-review blocking
title: Block a task on PR review, watch for reviews, auto-resume the agent
size: L
requires: [F12, E01, H07]
novel: true
---

## What
A task can be **blocked on the review of one or more specific GitHub pull
requests**. The blocker is a `blocked-by-pr` row in the existing git-synced
`relations.md` (from = task, to = PR url) — no new file. The app shows a
dedicated **"PRs" view** of every PR currently waiting for review, and runs a
**background poll** (default every 5 minutes, **only while the app window is
active**) that asks GitHub — via the `gh` CLI — whether each tracked PR has been
reviewed by a human.

When a tracked PR gets its **first human review** (any submitted review —
Approve, Request-changes, or a plain commented review — from someone other than
the PR author, excluding bots), the app reacts per blocking task:

- **The task already has a live terminal** → just fire a desktop notification
  ("PR reviewed — check it") so the human notices; don't disturb the running
  session.
- **The task has no live terminal** → open the task's terminal exactly as F12
  does (types `claude` + the work-on-task prompt, moves to IN PROGRESS), and
  appends an **extra prompt**:

  > PR &lt;url&gt; is reviewed. Check everything and identify next steps. Ask me
  > for approval.

One task can be blocked by **several** PRs, and one PR can block several tasks.
Blockers are added and removed from the PRs view (paste a PR URL, pick the task);
there is no per-task-panel editing for this relation.

### State model
- **`relations.md`** (git-synced, the existing typed-relations table): the
  durable blocker rows, as a new `blocked-by-pr` relation type (from = task,
  to = PR url). No separate file — PR blocking is just another relation.
- **`.pr-status.json`** (machine-local, gitignored, like `.llm-status.json`):
  the transient per-PR poll result — review state (`pending` / `reviewed` /
  `error`), the human reviewers seen, latest review time, PR title + open/merged
  state, last-checked time, and the last review time already **handled** (so a
  re-poll or an app relaunch never re-fires the same review). The review
  **baseline** also lives here: on first sight a PR's existing reviews are
  recorded as already-handled, so only a review that lands *after* you start
  watching fires — no `addedAt` column is needed in the synced file.

### The PRs view
A top-level view (sidebar nav next to Board / Terminal / Prompts) listing every
`pr-blocks.md` row: PR title + link (opens in the browser), the owning task
(links to it), review state badge (⏳ awaiting review / ✅ reviewed by _names_ /
⚠️ error / merged·closed), and last-checked time. It has an **add** control
(task picker + PR-URL field), a **remove** per row, and a **"Check now"** button
that forces an immediate poll instead of waiting for the timer.

## Why it exists
The whole app is "point an agent at a task, then get out of the way." The most
common *external* wait in that loop is **"I opened a PR, now I'm waiting on a
reviewer."** Until now that wait was invisible to TaskMan — the task just sat in
a column and the human had to remember to check GitHub. This closes the loop:
the app watches the PR, and the moment a human weighs in it either taps you on
the shoulder (if you're already in the terminal) or **re-summons the agent with
the review context**, so "address the review comments" starts itself. It is the
`waiting`→`attention` transition (H10) applied to the one wait the agent can't
see from inside the sandbox.

## Acceptance criteria (EARS)
- The system shall let a task be blocked by one or more GitHub PR URLs, persisted
  as `blocked-by-pr` rows (from = task, to = PR url) in the git-synced
  `relations.md`.
- The system shall provide a dedicated PRs view listing every tracked PR with its
  owning task, review state, reviewers, and last-checked time.
- The system shall let the user add a PR blocker (task + URL) and remove a PR
  blocker from the PRs view.
- While the app window is **active (focused)**, the system shall poll each
  tracked PR's review state on a fixed interval (default 5 minutes); it shall not
  poll while idle in the background/overnight, and shall poll immediately on
  regaining focus, when a blocker is added, and on demand ("Check now").
- The system shall treat a PR as *reviewed* when a review was **submitted by a
  non-author, non-bot user after the blocker's `addedAt` baseline**; Approve,
  Request-changes, and commented reviews all qualify.
- When a tracked PR becomes reviewed and a blocking task **has a live terminal**,
  the system shall fire a desktop notification and shall **not** open a new
  terminal.
- When a tracked PR becomes reviewed and a blocking task **has no live terminal**,
  the system shall open the task's terminal (F12 auto-start + IN PROGRESS) and
  append an extra prompt naming the reviewed PR and asking for next steps +
  approval.
- The system shall fire the reviewed reaction **once** per review event: a later
  poll, or an app relaunch, shall not re-fire for a review already handled.
- When a PR cannot be queried (no `gh`, not authenticated, network/HTTP error),
  the system shall surface an `error` state for that PR and keep polling the rest.
- The `blocked-by-pr` relations shall be git-synced (in `relations.md`); the
  `.pr-status.json` poll cache shall be machine-local and gitignored.

## Build notes
- **Reuse the F12/K01 terminal path.** The reviewed→open reaction is the existing
  `termOpen({ taskId, autoClaude:true })` route (so IN PROGRESS + `claude` + the
  work-on-task prompt all come free); the only new plumbing is an optional
  `extraPrompt` threaded into `autoStartClaude`, appended to the standard prompt
  in the **same** submission (one Claude turn, not two).
- **"Live terminal" test lives in main.** Query the tmux session list for a
  `task-<id>*` session; that decides notify-only vs open. Deciding in main avoids
  a renderer round-trip and is correct even for restored-but-not-yet-attached
  tabs.
- **`gh` is the interface.** `gh pr view <url> --json title,state,author,reviews`.
  Filter `reviews[]` to `author.login !== pr.author.login`, drop `*[bot]` logins,
  keep submitted reviews (`submittedAt` present), and compare the newest
  qualifying `submittedAt` against the baseline / last-handled time.
- **Fire once + baseline.** Persist the handled review timestamp in
  `.pr-status.json`; only a strictly-newer qualifying review re-arms. On first
  sight of a PR, seed handled = its newest existing review so pre-existing
  reviews don't fire. Same "don't re-notify" discipline as the E01 notifier and
  the H07 status notifications.
- **Poll cadence + active-gating.** A single `setInterval` in main (default
  5 min) whose tick **no-ops unless the window is focused** — so an app left open
  overnight spends no cycles or API calls. The main-process `focus` event kicks
  an immediate catch-up poll; adding a blocker and the "Check now" IPC poll
  unconditionally. Runs only while the desktop app is open (opening a terminal is
  a renderer operation) — same constraint as K01 autorun.
- **Guardrail alignment.** Auto-opening a terminal *starts an autonomous agent*,
  but here it is gated on a real external event (a human reviewed the PR) and the
  appended prompt explicitly ends with "Ask me for approval" — the agent
  investigates and proposes, the human still decides. No unguarded auto-apply.
- **Scope.** PR blockers are managed only in the PRs view; the task-to-task
  blocking UI (D02) filters to the `blocks` type and ignores `blocked-by-pr`, so
  the two coexist in `relations.md` without cross-talk. A small card badge for
  "has PR awaiting review" is optional polish.
