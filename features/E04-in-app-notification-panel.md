---
id: E04
epic: E — Native notifications
title: In-app notification panel (hideable right sidebar; starred terminal needs you → jump; task entered Today → auto-clears)
size: M
requires: [E01, F22, H08, H09, Q02, Q05, Q08]
novel: true
---

## What
A **notification panel on the right edge of the window**. Hidden, it takes no
space at all: a **"🔔 Notifications N" row in the left sidebar** (next to Today)
shows the count and opens it, and that row is absent while there is nothing to
show. Shown, it is a sidebar on the right listing notifications, newest first. Each notification has an icon, a one-line title, a
short detail line, the time it was added, its action button(s) and an **×** that
dismisses it. The header has **Clear all** and a button that hides the panel
again.

Notifications are **derived from live state**, not fired once and forgotten. Each
source produces the set of things that currently deserve a notification, each
with a stable key; the panel reconciles against that set:

- a key that appears and isn't dismissed → a notification is added;
- a key that disappears → its notification is **removed automatically**, and any
  dismissal of that key is forgotten (so if the condition comes back later, it
  notifies again);
- a key that stays → its notification stays (title/detail refreshed); a dismissed
  one stays dismissed for as long as the condition holds.

Two sources to start:

1. **A starred terminal needs you.** A terminal that is open, starred (F22), not
   hibernated, and whose per-terminal status (H08) is `attention` (⚠️) or
   `approval` (🔐). Key: `attn:<session name>`. Action: **Go to terminal** — brings
   the terminal view forward with that tab active (as H09's jump does). The
   condition is suspended while that terminal is the one on screen (terminal view
   active, that tab selected): looking at it handles it, so the notification
   clears, and it comes back if the user leaves without answering.
2. **A task entered Today.** A row of the Today view (Q02) — a task occurrence, a
   held (snoozed) occurrence of a repeating task (Q03), or a subgoal with a due
   (Q05) — whose Due falls **before the end of the current local day** (Today's
   *Overdue* and *Today* groups) and that the Today priority filter (Q08) shows.
   Key: `today:<task id>` for a one-off task, `today:<task id>@<occurrence key>`
   for an occurrence of a repeating task, `today:<task id>#<subgoal id>` for a
   subgoal. Action: **Open** — opens the task (its panel) on the board. Done,
   snooze past midnight, moving it to DONE/CLOSED, clearing its Due, or a filter
   change that hides it all take it out of Today and so remove the notification.

## Why it exists
Desktop notifications (E01, H07, Q04) are fire-and-forget: missed while away from
the screen, gone from Notification Center after a while, and they never retract
themselves when the thing they announced is no longer true. With a dozen agent
terminals and a Today list, the user wants one place inside the app that answers
"what needs me right now" — and stays accurate on its own: a starred agent that
was answered, or a chore that was ticked off, should not leave a stale card
behind. The ★ filter already says which agents matter most; Today already says
what is due — the panel turns their *transitions* into an inbox.

> *"A right-side panel/tab that can be hidden, holding notifications, each with
> an × and possibly action buttons. Starred terminal needs my attention → button
> takes me to it. Task entered Today → removed automatically once it leaves."*

## Acceptance criteria (EARS)
- While the panel is hidden, the system shall take no space for it at the right
  edge of the window (no tab, strip or column).
- While there is at least one notification, the system shall show a
  "🔔 Notifications" row in the left sidebar with the count as a badge; while
  there are none, it shall not show the row.
- When the user clicks the sidebar row, the system shall show the panel as a
  sidebar at the right edge; when the user clicks the row again or the panel's
  hide button, the system shall hide it.
- While the panel is shown and the last notification goes away, the system shall
  keep it shown with an empty-state line until the user hides it.
- The system shall remember whether the panel is shown across restarts
  (machine-local view state).
- The system shall list notifications newest first, each with an icon, a title, a
  detail line, the time it was added, its action buttons and an × button.
- When the user clicks × on a notification, the system shall remove it and shall
  not add it again while its condition keeps holding.
- When the user clicks **Clear all**, the system shall dismiss every current
  notification, as if × were clicked on each.
- While a terminal is open, starred, not hibernated, its status is `attention` or
  `approval`, and it is not the terminal currently on screen, the system shall
  keep one notification for it, titled with the task (and subgoal) of the tab and
  detailed with the status message.
- When the user clicks **Go to terminal**, the system shall show the terminal view
  with that terminal active.
- When a terminal's notification condition stops holding (status changes, the
  terminal is shown, unstarred, hibernated or closed), the system shall remove
  its notification without user action.
- While a Today row (task occurrence, held occurrence or subgoal due) is due
  before the end of the current local day and passes the Today priority filter,
  the system shall keep one notification for it, titled with the task id and
  title (and the subgoal title for a subgoal row) and detailed with its Due
  ("overdue since 09:00" / "due 18:00").
- When the user clicks **Open** on a Today notification, the system shall open
  that task on the board.
- When a Today row leaves that set (done, skipped, snoozed past today, moved to
  DONE/CLOSED, Due cleared, hidden by the filter), the system shall remove its
  notification without user action.
- When a condition stops holding, the system shall forget a dismissal of its key,
  so that the condition holding again later adds a new notification.
- The system shall re-evaluate every source whenever the task snapshot, the agent
  status map, the terminal list, the star set or the Today filter changes, and at
  least once a minute (Today rows cross midnight and become due by time alone).
- The system shall not reconcile a source until that source's data has loaded
  after launch, so a restart never clears persisted notifications or
  dismissals while the data is still empty.
- The system shall persist the current notifications (with the time each was
  added) and the dismissed keys machine-locally, so a restart keeps them and does
  not re-announce a dismissed item.
- The panel shall raise no desktop notification and play no sound of its own;
  existing native notifications (E01, H07, Q04) are unchanged.

## Decisions (defaults for the open questions)
- **Persistence / history:** current notifications and dismissed keys only, in
  machine-local view storage (like stars and tab order). No history of past
  notifications: once removed or dismissed, it's gone. Dismissed keys are pruned
  as soon as their condition stops holding, so the store stays small.
- **Hidden state:** a sidebar row, present only while there are notifications
  (an earlier always-present 28 px right-edge tab wasted a whole column of
  terminal width for one icon). The open panel stays docked (it narrows the
  content), not an overlay.
- **Badge:** the sidebar row shows the total count of current notifications
  (there is no unread/read distinction — the list *is* the to-do set, and every
  entry auto-clears when handled).
- **Attention auto-clear:** yes. A starred-terminal notification is removed as
  soon as the terminal leaves `attention`/`approval` — answering the agent is
  handling it — and also while the user is looking at that terminal.
- **`approval` counts as needing you**, alongside `attention` (both are "the agent
  is blocked on the user"; see H12).
- **"Entered Today" means due by the end of today** (Overdue + Today groups), not
  every row with any future Due — otherwise a task due next month would sit in
  the inbox for weeks. A one-off task snoozed to later the same day keeps its
  notification (its key is the task id); a repeating task's next occurrence is a
  new notification.
- **First launch:** everything already in the set becomes a notification (there
  is no "baseline" suppression) — the inbox should reflect what needs you now.
- **No ordering by priority**; newest first. The panel is not filtered by the
  category/★ terminal filter beyond the star itself.

## Build notes
- Pure logic in a shared module: `reconcile(prev, current, now)` takes the
  persisted `{items, dismissed}` and the sources' current `Map<key, {kind,
  title, detail, target}>`, returns the next state. Unit-test add / keep /
  refresh / auto-remove / dismiss-sticks / dismissal-forgotten.
- Source builders are pure too: `attentionSources(terminals, starred, status,
  hibernated, onScreen)` and `todaySources(tasks, held, subgoalDues, priority,
  now)` — the latter reuses Today's own occurrence/filter helpers so the two can
  never disagree.
- "Loaded" gating: track, per source, whether its first real data has arrived
  (snapshot loaded, status map seeded, terminal list seeded); pass only loaded
  sources to `reconcile`, and leave the other kinds' items untouched.
- Renderer only; no IPC, no files. Persist with `localStorage` under one key;
  wrap reads/writes in try/catch.
- Layout: the panel is the rightmost grid column, after the task side panel; while
  hidden it renders nothing there (the reconciler still runs). The shown/hidden
  flag and the count live in the app store so the sidebar row can read and
  toggle them.
