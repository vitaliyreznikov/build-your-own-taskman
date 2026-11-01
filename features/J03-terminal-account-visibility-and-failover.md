---
id: J03
epic: J — Claude account usage
title: Live account per terminal, move between accounts, quota failover
size: M
requires: [J02, H03, F22]
novel: true
---

## What
Builds on J02's two Claude accounts:

- **Live account tag in the window title.** The app's window title ends with
  the active terminal's account tag, e.g.
  `Taskman — I244: Improve Taskman · terminal [W]`. The tab labels stay short
  and carry no tag: users keep many terminals open, and a tag on every tab
  widens the strip. The tag reflects the account the running Claude *actually*
  uses, as Claude reports it through the status hook. That covers terminals
  opened before J02 and terminals switched by hand
  (`export CLAUDE_CONFIG_DIR=…`). When no Claude is running, the tag falls back
  to the account the terminal was launched with.
- **Hover details.** The tab's hover text puts one fact per line: the task id
  and title, the subgoal id and title (for a subgoal terminal), the Claude
  account, then the agent status.
- **Move to another account.** Right-clicking a tab offers "Move to <account>"
  for each other account. It works only while that terminal's Claude is idle.
  It exits Claude, switches the terminal's account, and resumes the same
  conversation on the new account.
- **Quota failover at launch.** When a terminal starts and the account the rule
  picked is out of quota while another account still has headroom, the terminal
  starts on the other account instead, and says so.

## Why it exists
J02 decides the account at launch, but the user can't see what a terminal is
running on once it exists. Old terminals have no record, and a hand switch is
invisible to the app. Two pools are only useful if the user can tell which one
each agent is draining and move work between them. And the point of having a
second pool is to keep working when one runs dry. A launch that lands on an
exhausted account just stalls on Claude's limit message, when the other account
could have taken it.

## Acceptance criteria (EARS)
- Where two or more accounts are configured, while the terminal view shows a
  terminal, the system shall end the window title with the short tag of that
  terminal's account (`… · terminal [W]`), and shall not add the tag to the tab
  labels.
- The account shown shall be the one the terminal's Claude session reports
  through its lifecycle hook, while a session is reporting.
- When no Claude session is reporting in a terminal, the system shall use the
  account the terminal was launched under, if known, and append no tag
  otherwise.
- When the user hovers over a terminal tab, the system shall show, each on its
  own line, the task id and title, the subgoal id and title (subgoal terminals
  only), the Claude account name (noting when it is only the launch account),
  and the agent status.
- When the user right-clicks a terminal tab and two or more accounts are
  configured, the system shall offer to move the terminal to each account other
  than its current one.
- When the user chooses to move a terminal whose Claude is idle, the system shall
  exit Claude, set the terminal's account to the chosen one, resume the most
  recent Claude session of that terminal on the chosen account, and remember
  the new account for restore.
- If the terminal's Claude is busy (working or waiting on a permission prompt)
  when a move is chosen, then the system shall refuse the move and say why.
- When the system launches a Claude terminal without an explicit account pick,
  and the account selected by J02's rule has a usage window at or above the
  failover threshold, the system shall launch on the first other configured
  account that is below the threshold in all its windows.
- When a launch fails over to another account, the system shall tell the user
  which account was skipped and why.
- If every configured account is at or above the threshold, or usage cannot be
  read, then the system shall launch on the account J02's rule selected.
- The system shall never fail over a launch whose account the user picked
  explicitly.

## Build notes
- Subgoal titles for the hover text come from the task's document. Read it
  lazily on first hover and cache it briefly, rather than loading every
  document up front.
- **Source of truth for the tag is the hook**, not tmux. Claude Code's lifecycle
  hooks (H03) run inside the Claude process, so they see its real
  `CLAUDE_CONFIG_DIR` — including one exported by hand in the shell, which
  `tmux show-environment` cannot see. Add `configDir` ('' = default dir) to the
  per-terminal status entry the hook already writes. The app maps it to an
  account by comparing normalised paths. An account's tag comes from config
  (`tag`), else the first letter of its name. The status entry is cleared when
  Claude exits, which is exactly when the fallback (J02's launch record) should
  take over.
- **Move = exit + resume.** A running Claude can't change accounts. Send
  `/exit`, wait for the shell, then type
  `export CLAUDE_CONFIG_DIR=<dir>` (or `unset CLAUDE_CONFIG_DIR` for the
  default account) and `claude --resume <session id>`. The session id is the
  terminal's most recent one from the I01 history. Both accounts share one
  `projects/` directory (J02's setup), so the transcript is found either way.
  Also update the tmux session environment, so a later new shell or pane in the
  same session keeps the account. Gate on idle using the same signal F10's
  "exit Claude first" check uses.
- **Failover threshold** — a window counts as exhausted at 100% used
  (`utilization ≥ 1`), configurable as `failoverAtPct` in the accounts config.
  Check the 5-hour and weekly windows. Leave the per-model weekly cap out: it
  limits one model, not the account. Cache usage reads for about a minute in
  the main process, so a burst of launches (reboot restore, several autoruns)
  doesn't fan out into usage calls. A usage read error means "unknown", never
  "exhausted".
- **Apply failover after J02's resolution**, to everything except an explicit
  pick: category rule, resume, restore. A resume can change accounts safely,
  because the transcript is shared.
- Surface a failover as an in-app toast plus a log line, e.g.
  "Bolt is out of quota (5h 100%) — started I244 on Personal".
- The over-budget Claude-vs-Devin check (F15) runs on the account chosen after
  failover.
