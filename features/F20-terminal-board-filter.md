---
id: F20
epic: F — In-app terminals
title: Filter terminal tabs by board
size: S
requires: [F06, A05]
novel: false
---

## What
A **Terminals** filter section in the sidebar (left pane) lists every board that
currently has at least one open terminal, each with a checkbox. Unchecking a
board hides all of that board's terminal tabs from the Terminal-view strip;
re-checking shows them again. A terminal's board is the board of the task it is
linked to (honouring a P05 re-bind); terminals with no linked task — or one whose
board is unknown — fall into a "— no board —" group with its own checkbox. The
selection is a local UI preference that persists across restarts. By default
every board is shown (nothing hidden).

## Why it exists
With many terminals open across work and personal life, the strip is one long
undifferentiated row. The user wants to focus: hide the work board ("bolt" =
`job`) to see only personal/brootto terminals, or hide everything except one
board. A per-board checkbox in the left pane does both — uncheck `job` to hide
work, or uncheck every board but one to see only that one.

> *"I need filters in the left pane to hide bolt or all other tabs from the
> terminal list."*

## Acceptance criteria (EARS)
- The system shall show, in the sidebar, a checkbox for each board that has one
  or more open terminals, plus a "— no board —" entry when any open terminal has
  no resolvable board.
- When the user unchecks a board, the system shall hide every terminal tab whose
  linked task belongs to that board from the Terminal-view tab strip.
- When the user re-checks a board, the system shall show that board's terminal
  tabs again.
- When the currently-active terminal is hidden by a filter change, the system
  shall move focus to the first still-visible terminal (or none if all are
  hidden), so a hidden pane is never left showing.
- The Tab-key "next attention" cycle and the "needs you" jump shall consider only
  visible terminals.
- When the board filter changes, the system shall persist the hidden-board set so
  it is restored on the next launch.
- The system shall not create, rename, kill, or restart any tmux session when a
  board is hidden or shown — the filter is view-only.
- A terminal's board shall follow a P05 re-bind (the board of the task it is
  re-bound to), matching the id shown on its tab.

## Build notes
- A terminal's board is derived, not stored: map the terminal's effective task id
  (`rebinds[name] ?? taskId ?? parseTaskTerm(name).taskId`) through the loaded
  task list (`view.tasks`, each task's `board`). Terminals with no match group
  under a `NO_BOARD` sentinel labelled "— no board —".
- Persist the hidden set the same way F16 dividers are persisted — a small local
  `localStorage` list (dividers/terminal filters are pure view preferences, not
  tmux state, so they can't ride `.terminals.json`). Store hidden boards (not
  shown), so a newly-appearing board defaults to visible.
- In the Terminal view, build the strip's unified item list from the **visible**
  terminals only; the pane render loop can keep all panes mounted (xterm
  survival) since inactive panes are already `display:none` — the one guard
  needed is re-homing `activeTerminal` when the filter hides it.
- The sidebar section is derived live from `terminals` + `view.tasks`; render it
  only when at least one terminal is open, and skip a board's checkbox once it has
  no terminals so the list tracks reality. A board hidden while it had terminals
  but now has none simply drops from the list (its entry in the hidden set is
  harmless and reappears checked-off if the board returns — acceptable).
- Keep the existing TerminalStates tally counting **all** terminals: the tally is
  a global "how much is running" signal; the filter is only about which tabs the
  strip shows.
