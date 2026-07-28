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
- Where the subgoal's text is known, the system shall show the subgoal id and
  its text; where only the id (or the session-name slug) is known, the system
  shall show that alone, without a dangling separator.
- Where the subgoal text is long, the system shall truncate it so the title
  stays readable in a title bar and window switcher.
- When the active view is a task-level terminal or a task, the system shall
  produce the same title as B13 — no empty subgoal segment.

## Build notes
- Extend the existing single title-computing effect (B13); do not add a second
  writer of `document.title`.
- The subgoal id and text ride on the terminal tab when the terminal was opened
  from the document view; a terminal rediscovered from tmux after a restart only
  has the slugified id parsed out of its session name (O09) — fall back to it.
- Use the same fallback chain the terminal tab label uses, so the tab and the
  window title never disagree about what a terminal is working on.
