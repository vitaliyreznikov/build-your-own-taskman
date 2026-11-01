---
id: J02
epic: J — Claude account usage
title: Two Claude accounts — per-category default, override, both limits shown
size: M
requires: [J01, A14, F12, F14, I02]
novel: true
---

## What
TaskMan can drive Claude Code under **two signed-in Claude accounts** on one
machine and picks one for every Claude terminal it launches:

- **Default by category** — a machine-local mapping says which account a task's
  category uses (e.g. `bolt` → account 1, every other category → account 2).
- **Per-terminal override** — the agent chooser that appears when a terminal is
  launched offers the account too, preselected from the category rule; the pick
  applies to that one terminal only.
- **Headless launches follow the rule** — autorun (task and subgoal), `taskman run`
  task workers and reboot-restore pick the account the same way, with no prompt.
- **Both limits visible** — the J01 usage widget shows one block per configured
  account, each with its own 5h / weekly / per-model rows, labelled with the
  account's short name.

With one account configured nothing changes: no account picker, one widget block.

## Why it exists
One Claude subscription runs out. A user with a work subscription and a personal
one wants work tasks to burn the work quota and everything else the personal
one, without remembering to switch logins (Claude Code has one login per config
directory, and `/login` in a shared directory flips *every* session). Seeing both
quotas side by side shows which pool has headroom before launching heavy work, and
the override lets the user deliberately move one job to the other pool.

## Acceptance criteria (EARS)
- Where two or more accounts are configured, when the system launches a Claude
  terminal for a task, the system shall run Claude under the account mapped to
  the task's category, or under the default account when the category has no
  mapping.
- Where two or more accounts are configured, when the user launches a terminal
  through the agent chooser, the system shall offer an account choice
  preselected from the category rule, and shall launch that one terminal under
  the chosen account.
- When the system launches a terminal without user interaction (autorun, a
  control-API task worker, a reboot restore), the system shall apply the
  category rule without prompting.
- When a terminal is restored after a reboot or app restart, the system shall
  reopen it under the account it was launched with.
- When the user resumes a past session, the system shall resume it under the
  account the session originally ran on, falling back to the category rule when
  that is unknown.
- The system shall display usage limits (as in J01) for every configured account,
  each block labelled with that account's name.
- When one account's usage cannot be fetched, the system shall show that
  account's block as unavailable without affecting the other account's block.
- Where the over-budget check (F15) runs for a launch, the system shall evaluate
  it against the account that launch will use.
- Where only one account is configured, the system shall behave exactly as
  before this feature (no account picker, single usage block, unchanged launch
  environment).
- The system shall label each Claude terminal tab with the account it runs under
  when two or more accounts are configured.

## Build notes
- **Mechanism is Claude Code's config directory.** Claude Code reads its whole
  state — login, settings, hooks, transcripts — from `$CLAUDE_CONFIG_DIR`
  (default `~/.claude`, with `~/.claude.json` beside it; with the env var set the
  JSON is `$CLAUDE_CONFIG_DIR/.claude.json`). An account *is* a config dir. The
  default account keeps the default dir and must run with `CLAUDE_CONFIG_DIR`
  **unset** — setting it, even to `~/.claude`, changes where Claude Code looks
  for its credentials. Every other account sets it per terminal (tmux
  `new-session -e CLAUDE_CONFIG_DIR=…`), so concurrent sessions on different
  accounts never interfere.
- **Credentials on macOS.** The default dir's OAuth token is in the login
  Keychain under service `Claude Code-credentials`; a non-default dir stores it
  under `Claude Code-credentials-<first 8 hex of sha256(dir path)>`. The usage
  reader computes that name per account, and allows an explicit
  `keychainService` override in config in case the scheme changes.
- **Config** — machine-local, gitignored (paths are per machine):
  ```json
  {
    "accounts": [
      { "id": "a1", "name": "Bolt", "configDir": null },
      { "id": "a2", "name": "Personal", "configDir": "~/.claude-acct2" }
    ],
    "default": "a2",
    "byCategory": { "bolt": "a1" }
  }
  ```
  `configDir: null` = the default dir. Missing/malformed file = single account,
  pre-J02 behaviour.
- **The second config dir starts empty**, without the user's settings, hooks
  (which drive the H-epic status icons), skills or memory. Share them by
  symlinking `settings.json`, `CLAUDE.md`, `skills/`, `agents/`, `plugins/`,
  `commands/` and `projects/` (transcripts + memory) to the default dir, so both
  accounts behave identically and either can `--resume` any session. Only the
  login (Keychain entry + `.claude.json`) stays per account. User-scope MCP
  servers live in `.claude.json`, so copy its `mcpServers` once. Ship this as a
  setup script; logging in (`CLAUDE_CONFIG_DIR=<dir> claude` then `/login`) is
  the user's step.
- **Remember the account per terminal** (alongside the restore record of F14 and
  the session history of I01) so restore and resume use the same account even if
  the category mapping or the task's category changed since.
- **Usage widget** — fetch accounts independently (`Promise.allSettled`), keep
  J01's activity-gated refresh and fast local tick, render one labelled block
  per account in config order.
