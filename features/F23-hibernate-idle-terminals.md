---
id: F23
epic: F — In-app terminals
title: Hibernate idle agent terminals (tab stays, click wakes the same conversation)
size: M
requires: [F14, H08, I01, J03]
novel: true
---

## What
An agent terminal that has sat idle for hours still holds a full Claude Code
process — plus every MCP server that session started (a browser driver, a database
client, …). With dozens of tabs open that is gigabytes of RAM and swap for
conversations nobody is looking at.

**Hibernation** exits the idle agent but keeps everything the user sees: the tab
stays in the strip (with a 💤 marker), the tmux session and its scrollback stay,
and the shell stays in the conversation's directory. The conversation id is
recorded first. When the user next **views** the tab, the app types
`claude --resume <id>` into it and the same conversation comes back.

Three ways in:
- **Automatic** — a periodic sweep hibernates every terminal that is idle past a
  threshold (default 3 h) and is safe to stop (see criteria).
- **Manual** — a tab's right-click menu has "💤 Sleep"; the tab bar shows
  "🤖 N · 💤 M" (live agents, sleeping tabs), and clicking it sleeps every safe
  terminal that has been quiet for ~15 minutes.
- **Restore asleep** — the F14 reboot restore recreates every terminal *asleep*
  (plain shell in the conversation's directory, conversation id recorded) instead
  of starting dozens of agents at once; each one wakes on first view.

## Why it exists
On a 32 GB machine, 67 restored-but-untouched agent sessions (~16 GB RSS, swap
13 of 14 GB, load average 400) made the whole computer unusable; 49 of them hadn't
been touched in over a day. Killing the sessions would lose the tabs and the
user's place; hibernating frees the memory and keeps both.

> *"Компьютер тормозит, когда открыто много терминалов. Нужно, чтобы простаивающие
> терминалы почти не занимали память, но вкладки оставались: клик по вкладке →
> сессия возвращается."*

## Acceptance criteria (EARS)
- The system shall consider a terminal **safe to hibernate** only while all hold:
  its foreground process is Claude; its agent status is absent, `done`, or
  `attention` (never `working`, `waiting` or `approval`); the agent's conversation
  id is known; the agent's input box is visible and holds no typed text (a dimmed
  auto-suggestion does not count as text); and the agent shows no running
  background task.
- The system shall consider a terminal **idle** when neither its pane output nor
  the user's viewing of it has happened within the threshold.
- While a terminal's agent waits for the user (`attention`), the system shall
  require the longer attention threshold (default 24 hours) instead, and shall
  keep the question with the record so the tab shows 💤⚠️ and its hover text
  repeats the question. *(The user: an unanswered question loses nothing on
  resume — the conversation still holds it.)*
- While automatic hibernation is enabled, the system shall periodically hibernate
  every terminal that is safe and idle past the configured threshold (default
  3 hours), one at a time.
- The system shall never hibernate the terminal the user is currently viewing.
- When the system hibernates a terminal, it shall record the conversation id,
  directory and time **before** exiting the agent, then exit it the same way a
  user would (`/exit`), keeping the tmux session, scrollback and tab.
- If the agent has not exited within 15 seconds, the system shall drop the record
  and leave the terminal as it was.
- While a terminal is hibernated, the system shall show 💤 on its tab instead of
  the agent-status icon, and say "asleep — view to wake" in its hover text.
- When the user views a hibernated terminal (tab click, task-panel terminal, or a
  focus by autorun / the control API), the system shall type
  `claude --resume <id>` into it and clear the record.
- If a hibernated terminal's agent is started by other means (the user typed
  `claude`), the system shall drop its record rather than start a second agent.
- When the user picks "💤 Sleep" on a tab, the system shall hibernate that
  terminal if it is safe, ignoring the idle threshold, and otherwise say why not.
- The system shall show "🤖 N · 💤 M" — live agent terminals and hibernated
  terminals — in the tab bar; when clicked, it shall hibernate every safe,
  non-viewed terminal idle for at least 15 minutes.
- When the user restores terminals after a reboot (F14), the system shall create
  each one hibernated in its conversation's directory, with no agent started,
  and wake each on first view.
- When a terminal is closed (killed), the system shall drop its record.
- The system shall record in the F14 snapshot each terminal's **own**
  conversation — its hibernation record, else its status entry — and fall back
  to the task's newest conversation only for a terminal that never reported one.
  *(Amended 2026-11-05: the per-task fallback alone resumed all 19 terminals of
  one task into the same conversation after a reboot.)*
- The system shall keep hibernation records and settings machine-local (never
  synced): `enabled`, `idleHours` and `attentionHours`, defaulting to on, 3
  and 24.

## Build notes
- **Idle signal:** tmux's `#{session_activity}` (epoch seconds of the last pane
  output) — an idle Claude at its prompt prints nothing. Combine with a
  renderer-reported "last viewed" time per session (the renderer pings on every
  view, which also serves as the wake trigger).
- **Conversation id:** Claude Code keeps a per-process record
  `<configDir>/sessions/<pid>.json` (`sessionId`, `cwd`) from the moment it starts;
  prefer it, then the status hook's id, then a `--resume <id>` in the process's
  arguments (older builds wrote no record). The hook alone misses every terminal
  restored after a reboot and not touched since — in practice most of them.
- **Empty input box:** `capture-pane -e -p`, find the last line starting with `❯`
  and the lines up to the box's bottom border; any visible character outside a
  dim (SGR 2) span is a draft → not safe. No `❯` line (a dialog, a permission
  prompt) → not safe.
- The SessionEnd hook clears the status entry when the agent exits, which is why
  the id must be written to the hibernation record first.
- The shell stays where the agent was launched, so `claude --resume <id>` finds
  the transcript (Claude keys history by cwd) and the terminal's account env
  (J02/J03) is unchanged.
- MCP servers are children of the agent process, so they exit with it — that is
  most of the saving.
- Records: `.hibernated.json` `{ "<session>": { sessionId, cwd, at } }`;
  settings: `.hibernate.json` `{ enabled, idleHours, attentionHours }`. Prune records whose session
  is gone at startup.
