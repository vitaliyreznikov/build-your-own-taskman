---
id: F21
epic: F — In-app terminals
title: Persistent default terminal provider (Claude or Codex)
size: M
requires: [F12]
novel: false
---

## What

TaskMan adds a sidebar setting named **Default provider** with two choices:
**Claude** and **Codex**. The choice is stored locally on the machine and applies
to every newly created task, subgoal, and new-task terminal. A fresh install (or
a missing/corrupt preference file) continues to use Claude.

Selecting Codex starts its interactive CLI with the same work prompt F12 would
send to Claude:

```
codex --approve-for-me <work prompt>
```

The preference is intentionally a default, not a restriction: the existing
over-budget chooser can still offer an explicit provider override where it is
available. Resuming session history remains Claude-only because that history is
owned by Claude Code.

## Why it exists

> TaskMan previously assumed a Claude subscription was always available. When a
> user instead has Codex access, every normal terminal open fails before work can
> begin. A visible, persistent provider choice lets the app launch the coding
> agent the user currently has.

## Acceptance criteria (EARS)

- When the user selects **Codex** as the default provider, the system shall
  persist that selection locally and shall launch Codex for each subsequently
  created task, subgoal, and new-task terminal.
- When the user selects **Claude** as the default provider, the system shall
  persist that selection locally and shall launch Claude for each subsequently
  created task, subgoal, and new-task terminal.
- When the stored provider preference is absent or invalid, the system shall
  use Claude and shall not fail to start.
- When Codex is selected, the system shall give it the same task/subgoal/create
  prompt used by the Claude path and shall use Codex's normal interactive CLI.
- When a prior Claude session is resumed, the system shall continue to launch
  Claude regardless of the default provider.
- When the user changes the setting, existing terminals shall keep running their
  current provider and only terminals created afterwards shall use the new one.

## Build notes

- Store `{ provider: 'claude' | 'codex' }` in a machine-local, gitignored JSON
  file beside the existing theme preference. Read defensively and write
  atomically.
- Expose get/set IPC methods through the preload bridge. Keep the main process
  as the source of truth so all renderer launch paths can request the same
  default.
- Use a small segmented control in the sidebar settings area, matching the theme
  control. Reflect the saved value on startup and update it optimistically after
  a user click.
- Generalize the agent union, terminal tab metadata, and launch function from
  `claude | devin` to `claude | codex | devin`. Codex starts with
  `codex --approve-for-me <prompt>`; escape the prompt with the existing shell
  argument quoting helper. There is no Codex session-history resume path in this
  feature.
- Existing Claude-budget behavior may remain scoped to Claude and Devin. Normal
  default-provider opens must not query Claude account usage before starting
  Codex.
