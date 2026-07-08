---
id: H06
epic: H — LLM / agent status via Claude hooks
title: Permission-prompt to attention (automatic)
size: S
requires: [H03]
novel: false
---

## What
When the agent hits a point where it needs the human — a permission prompt, an
`AskUserQuestion`, an `ExitPlanMode`, or a Claude Code notification — the hooks
set the task's status to ⚠️ **attention automatically**, without the agent having
to emit a marker. The moment an agent is blocked on the user, the board shows it.

## Why it exists
The single most important guarantee of the supervision layer is that nothing
needing the human is ever silent. An agent stopping to ask a question is the
canonical "needs you" moment — and it must surface even if the agent forgot to
write `⚠️ attention:`. Detecting it from the lifecycle event makes the guarantee
mechanical, not dependent on agent discipline.

> The author's rule, stated flatly: *"a question = attention, not done."* And more
> broadly: *"Everything for me — should be attention."* A permission prompt is
> exactly that — something the agent needs from him.

## Acceptance criteria (EARS)
- When a pre-tool hook fires for `AskUserQuestion` or `ExitPlanMode`, the system
  shall set the task's status to attention automatically.
- When a Claude Code notification (e.g. a permission prompt) fires, the system
  shall set the task's status to attention.
- The system shall set attention on these events without requiring an explicit
  marker in the agent's text.
- The system shall not overwrite a fresh done status with a generic idle
  notification (the idle-notification guard).

## Build notes
- Wire this into the pre-tool and Notification hooks in `claude-hook.mjs`;
  `AskUserQuestion` and `ExitPlanMode` are the two tool signals that mean "blocked
  on the human."
- Add the idle-notification guard: an idle/generic notification must not stomp a
  just-written done status with a vague attention — this was a real regression the
  reference implementation fixed.
- This event-driven attention coexists with marker-driven attention (H04); both
  land in the same status file (H02) and drive the same icon (H01).
