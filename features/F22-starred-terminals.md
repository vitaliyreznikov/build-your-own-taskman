---
id: F22
epic: F — In-app terminals
title: Starred terminals
size: S
requires: [F20]
novel: false
---

## What
Any terminal tab can be **starred** from its right-click menu ("☆ Star" /
"★ Unstar"). A starred tab shows a small ★ before its label; an unstarred tab
shows nothing extra, so the strip spends no space on the feature until it is
used. The sidebar's **Terminals** filter (F20) gets a **★ Starred** toggle at its
top: when it is on, the strip shows only starred terminals, from **every**
category — including categories the F20 filter hides. When it is off, the strip
is exactly the F20 strip.

## Why it exists
Terminals are spread across categories (bolt, personal, brootto…), and the F20
filter is per category. A handful of terminals matter right now regardless of
category; the user wants to mark them and see just those, across topics, in one
click.

> *"I need function to star any terminal and see all starred terminals across
> different topics."* — *"right click menu only and show star only for starred
> terminal — to save space."*

## Acceptance criteria (EARS)
- When the user right-clicks a terminal tab, the system shall show a menu with
  "☆ Star" for an unstarred terminal or "★ Unstar" for a starred one.
- When the user picks Star / Unstar, the system shall toggle that terminal's star.
- The system shall show a ★ marker on starred tabs only, and no star control on
  unstarred tabs.
- While at least one terminal is starred, the system shall show a "★ Starred"
  toggle with the starred count at the top of the sidebar's Terminals section.
- While "★ Starred" is on, the system shall show in the strip exactly the starred
  terminals, regardless of the F20 hidden categories.
- While "★ Starred" is off, the system shall show the strip as F20 defines it.
- The Tab-key "next attention" cycle, the "needs you" jump and the
  hidden-active-terminal re-homing shall use the same visible set as the strip.
- The system shall persist stars and the "★ Starred" toggle locally across
  restarts; stars are keyed by the tmux session name, so a terminal recreated by
  F14 restore keeps its star.
- When the user closes a tab (killing its session), the system shall drop its
  star.
- When the last starred terminal is unstarred or closed, the system shall turn
  the "★ Starred" mode off, so the strip never goes empty because of it.
- Starring shall not create, rename, kill or restart any tmux session.

## Build notes
- Store the starred set as a list of session names in `localStorage`, next to the
  F16 dividers and F20 hidden categories — pure local view preferences that
  can't ride `.terminals.json`. Do not drop a star in the generic
  "terminal went away" path: that also runs when the app restarts or the machine
  reboots, and the star should survive a restore. Drop it only on the explicit
  close (×).
- The visible-set predicate becomes
  `starredOnly ? starred.has(name) : !hiddenCategories.has(category(t))`; keep
  one predicate so the strip, attention cycle and re-homing can't disagree.
- A small positioned menu at the cursor is enough (close on outside click or
  Escape); no native menu needed.
