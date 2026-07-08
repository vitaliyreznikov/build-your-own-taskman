---
id: H03
epic: H — LLM / agent status via Claude hooks
title: Deterministic status via Claude Code lifecycle hooks
size: L
requires: [F02, H02]
novel: false
---

## What
Agent status is written deterministically by Claude Code lifecycle hooks
(`claude-hook.mjs`), which are the **sole writers** of `.llm-status.json` during a
session. Rather than trusting the agent to call an API, the hooks fire on real
lifecycle events (tool use, notification, stop) and derive the status by reading
the session transcript — the actual record of what happened this turn.

## Why it exists
Asking the agent to "remember to update its status" is unreliable; it forgets, or
reports stale phases. Hooks make the signal deterministic: they fire on events
the harness guarantees, and they read ground truth from the transcript. This is
what makes the whole supervision layer trustworthy enough to route a human's
attention across many parallel agents.

> The author imagined the agent self-reporting via an API — *"We need to provide
> LLM API to update such status … It starts from reading agents.md and know from
> it that it should update LLM state and notify via it when it's done."* Hooks are
> the robust realization of that intent: the status updates itself from the
> transcript instead of depending on the agent to remember.

## Acceptance criteria (EARS)
- The system shall register Claude Code lifecycle hooks that fire on tool use,
  notification, and stop events for a session.
- The hooks shall be the only writers of session status into `.llm-status.json`.
- When a hook fires, the system shall derive the status by reading the current
  turn from the session transcript (final message, current turn, user-turn
  count), not from agent-issued API calls.
- The hooks shall key status by the task identity exported into the terminal
  session (`TASKS_TASK_ID`), so status attaches to the right task.
- When no task identity is present, the hooks shall no-op rather than write
  status to the wrong place.

## Build notes
- `claude-hook.mjs` is the hook entry point; `transcript-tail.mjs` reads the
  current turn, the final assistant message, and the user-turn count from the
  transcript. Keep parsing here — downstream features (H04–H06) are consumers.
- Status keying depends on `TASKS_TASK_ID` being exported into the tmux session
  by the terminal's ptyManager (F02). Hooks only fire meaningfully when that env
  var is set — this is why F02 is a hard prerequisite.
- Derivation precedence: explicit markers in the answer text (H04) win; then
  fallbacks (H05); permission prompts force attention (H06). This file owns the
  event plumbing; the specific derivation rules live in the features that require
  it.
- Because hooks are the sole writer, the manual CLI (H01) and hooks must agree on
  the same file shape (H02).
