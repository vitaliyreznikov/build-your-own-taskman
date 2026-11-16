---
id: F25
epic: F — In-app terminals
title: Orchestrator tab mark
size: S
requires: [F22, J03]
novel: false
---

## What
Any terminal tab can be marked as an **orchestrator** from its right-click menu
("📡 Mark as orchestrator" / "📡 Unmark orchestrator"). An orchestrator is a
terminal whose agent mostly dispatches and supervises other workers instead of
doing the work itself. A marked tab looks different:

- a purple-tinted tab background, and a purple accent bar in place of the usual
  blue one while it is the active tab;
- a 📡 before its label;
- a 📡 in the window title right after the account tag
  (`Taskman [W] 📡 — I355: … · terminal`) while it is the active terminal;
- a line "📡 orchestrator" in the tab's hover tooltip.

The mark is only a view preference. It does not change the agent's prompt,
behaviour, permissions or account.

## Why it exists
With many terminals open, the one or two that coordinate the rest (a whole-task
terminal that hands subgoals to workers) look exactly like the workers. The user
wants to find the coordinator at a glance and know, once inside it, that this is
the coordinating session.

> *"A switch 'orchestrator / ordinary task' in the terminal's context menu, the
> orchestrator tab visually different — only the look changes."*

## Acceptance criteria (EARS)
- When the user right-clicks a terminal tab, the system shall show
  "📡 Mark as orchestrator" for an unmarked terminal or "📡 Unmark orchestrator"
  for a marked one.
- When the user picks that item, the system shall toggle the terminal's
  orchestrator mark.
- While a terminal is marked, the system shall show its tab with a purple-tinted
  background and a 📡 before the label; unmarked tabs shall look as before.
- While a marked terminal is the active tab, the system shall draw its accent bar
  in purple instead of the default active colour.
- While a marked terminal is the active terminal in the terminal view, the system
  shall put 📡 in the window title after the account tag and before the task id,
  leaving the rest of the title format unchanged.
- The system shall add a "📡 orchestrator" line to a marked tab's hover tooltip.
- When the user right-clicks a tab in a task's own terminal strip (the task
  panel), the system shall show the same item, and that tab shall show the same
  mark: one mark per session, shared with the main strip.
- The system shall persist marks locally across app restarts, keyed by the tmux
  session name, so a terminal recreated by F14 restore keeps its mark.
- When the user closes a tab (killing its session), the system shall drop its
  mark.
- Marking or unmarking shall not change the agent's prompt, the terminal's
  account, any permission, or any tmux session.

## Build notes
- Mirror F22 stars: a `Set<string>` of session names in `localStorage`, toggled
  from the same shared tab-menu component, dropped only on the explicit close (×)
  and never in the generic "terminal went away" path (that also runs on restart
  and reboot restore).
- Keep the colours as CSS tokens. The purple must stay distinct from the blue
  active bar and the gold ★, so a starred orchestrator still reads as both.
- Window-title consumers parse `I<digits>` (e.g. a time tracker), so the 📡 goes
  between the account tag and the em dash and the task-id part stays put.
