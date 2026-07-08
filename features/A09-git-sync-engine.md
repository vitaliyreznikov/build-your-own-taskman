---
id: A09
epic: A — Markdown data model
title: Git sync engine
size: M
requires: [A01]
novel: false
---

## What
A background sync engine (`sync.ts`) that commits the data files to git on a
debounce, so the whole task repo — notes, index, order, columns, boards, per-task
directories — is continuously versioned without the human ever running git by
hand. It combines a short debounce after changes with a periodic timer, producing
`auto-sync:` snapshot commits.

## Why it exists
If the source of truth is a pile of flat files, those files need to be durable and
recoverable, and their history is itself valuable (what changed, when, by which
agent). Automating commits means the human never loses work and can always diff or
roll back — the versioning safety net that makes "everything is just files"
trustworthy for daily, high-volume use.

## Acceptance criteria (EARS)
- When a data file changes, the system shall schedule a commit after a short
  debounce interval (~30s) rather than committing on every keystroke.
- The system shall also commit periodically (~every 4h) so long-idle changes are
  still captured.
- When it commits, the system shall stage and commit the task data files as an
  `auto-sync:` snapshot.
- When there are no changes, the system shall not create an empty commit.
- Sync shall run in the background without blocking the UI, and its outcome shall
  be reportable to the status indicator (A11).

## Build notes
- Live in the Electron main process; debounce (~30s after last change) plus a
  periodic (~4h) timer, coalescing bursts of edits into one snapshot commit.
- All the data files are in scope: notes (A01), `I<N>/` directories (A08), the
  index/order/columns/boards files, and later the synced JSON state files —
  except deliberately transient, gitignored files (e.g. the LLM status file).
- Commits are the `auto-sync:` noise you'll see in the log; keep the message
  prefix consistent so snapshots are easy to filter out when mining history.
- Pair with A10 (external mirror) and A11 (status indicator); the engine is the
  producer both of those consume.
