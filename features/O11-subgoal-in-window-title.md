---
id: O11
epic: O — Structured task documents (v2)
title: Subgoal in the window title
size: S
requires: [B13, O09]
novel: false
---

## What
When the active view is a subgoal terminal (O09), the OS window title names the
subgoal as well as the task — the task segment of B13's title is extended, not
replaced:

```
Taskman — I244: Claude session history · g3.5: wire the reaction ack · terminal
```

A task-level terminal keeps exactly the B13 title it has today.

## Why it exists
With per-subgoal terminals, several windows can be open on the same task and
B13's title makes them indistinguishable — every one reads `I244: … · terminal`.
The title bar is what external attribution tools (and the OS window switcher)
see, so it has to carry the finer-grained unit of work once that unit exists.

## Acceptance criteria (EARS)
- When the active view is a terminal scoped to a subgoal, the system shall
  include that subgoal in the window title in addition to the task id and title.
- The system shall resolve the subgoal's real id and text from the task's
  document, so a terminal it did not open itself (rediscovered from tmux after a
  restart, or started by another agent) still names its subgoal in full rather
  than showing the lossy session-name slug.
- Where the subgoal's text cannot be resolved, the system shall show the id (or
  the slug) alone, without a dangling separator.
- Where the subgoal text is long, the system shall truncate it so the title
  stays readable in a title bar and window switcher.
- When the active view is a task-level terminal or a task, the system shall
  produce the same title as B13 — no empty subgoal segment.

## Build notes
- Extend the existing single title-computing effect (B13); do not add a second
  writer of `document.title`.
- Fallback chain, widest source first: the task document (authoritative) → the
  id and text the terminal tab was opened with → the slugified id parsed out of
  the session name (O09).
- The slug is lossy (dots become underscores), so match a document subgoal by
  slugifying its id and comparing — never by trying to unslugify the name.
- Follow document changes: subgoal text is edited by the agent working the task,
  so the resolved label must be refreshed on the same file-change signal the
  document view uses, and keyed by session name so a stale answer is never shown.
- Use the same fallback chain the terminal tab label uses, so the tab and the
  window title never disagree about what a terminal is working on.
