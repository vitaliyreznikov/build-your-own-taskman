---
id: G02
epic: G — Prompt-parts library
title: Fast terminal insert (Cmd+K)
size: M
requires: [G01, F01]
novel: false
---

## What
A ⌘K quick-picker that inserts a prompt part (G01) straight into the current
terminal. The user hits ⌘K, chooses a snippet, and its text lands in the
terminal's input — no switching to the prompts view, no retyping.

## Why it exists
The prompt library only pays off if pulling a snippet is faster than typing it.
⌘K makes insertion a reflex inside the loop the user runs all day, so common
instructions reach an agent with two keystrokes.

## Acceptance criteria (EARS)
- When the user presses ⌘K in a terminal, the system shall open a picker of
  available prompt parts (from `prompts.md`).
- When the user selects a prompt part, the system shall insert its text into the
  active terminal's input.
- The ⌘K insert shall behave identically whether the terminal was opened from the
  task (F03 switcher) or from the standalone terminal view.
- When there are no prompt parts, the system shall still open the picker and
  indicate the library is empty.

## Build notes
- **Share one implementation across both entry points.** The reference version
  first shipped ⌘K working only from the terminal view and broke when the
  terminal was opened from a task, because the two entry points had duplicated
  code. The author was explicit that this is a smell to fix, not to work around:
  > *"this feature doesn't work when I open terminal from the task, not from the
  > terminal view. It means we use different code for two identical pieces which
  > is likely suboptimal. Investigate and find solution."*
  Route both surfaces through the same insert path (target the active terminal by
  its `TASKS_TERM_ID`) so there is a single code path to keep working.
- Insert into the terminal input rather than auto-submitting, so the user can
  edit before sending.
