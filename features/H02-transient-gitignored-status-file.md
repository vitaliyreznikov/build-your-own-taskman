---
id: H02
epic: H — LLM / agent status via Claude hooks
title: Transient, gitignored status file (.llm-status.json)
size: S
requires: [H01]
novel: false
---

## What
Agent status lives in a single transient file, `.llm-status.json`, that is
**gitignored** and never synced. It holds the current status and message per
status key (per task, later per terminal). It is deliberately separate from the
`Column` state that remains the source of truth for where a task is on the board.

## Why it exists
"What is the agent doing right now" is a live, ephemeral signal — not a fact
about the task worth committing to history. The board's `Column` (NEXT / IN
PROGRESS / …) is the durable truth; the status icon is a volatile overlay. Keeping
them in separate files, and keeping status out of git, prevents transient agent
chatter from polluting the synced record and from creating merge noise across
machines.

> The author's status protocol distinguishes momentary phase from durable state:
> *"when your phase changes add a line '🤖 working: <what you're doing>', and end
> every turn's final message with one marker line — '✅ done', '⏳ waiting: …' or
> '⚠️ attention: …'."* Those are moment-to-moment signals, not board state.

## Acceptance criteria (EARS)
- The system shall store current agent status and message in `.llm-status.json`,
  separate from the board's `Column` state.
- The status file shall be gitignored and shall never be included in the git /
  Obsidian sync.
- When the app restarts, the system shall read whatever status the file
  currently holds without treating a missing or stale file as an error.
- The system shall not derive or overwrite a task's `Column` from the status
  file (status and column are independent).

## Build notes
- One JSON file, keyed by status key. Start keyed per task; extend the key to
  per-terminal (`TASKS_TERM_ID`) under H08 without changing the file's role.
- Add `.llm-status.json` to `.gitignore` from the start — it is the one data file
  that must not travel with the repo.
- Writers are the hooks (H03) plus the manual CLI; readers are the card/tab UI
  (H01, H08). Keep it a plain read/write JSON store, not a schema-heavy format.
- Treat absence as "no status" everywhere, so a fresh checkout or a cleared file
  degrades gracefully to clean cards.
