---
id: H07
epic: H — LLM / agent status via Claude hooks
title: attention/done desktop notifications
size: S
requires: [H03, E01]
novel: false
---

## What
When an agent reaches ⚠️ attention or ✅ done, the app fires a native desktop
notification so the human learns of it without watching the board. Crucially,
there is **no notification when an agent starts** working, and the status icons
**never blink** — the app pushes only on the two events that actually warrant a
human's attention.

## Why it exists
Supervising many agents only works if the notifications are pure signal. A ping
on every start, or a blinking icon, trains the human to ignore the app — the
opposite of what it's for. Pushing exclusively on attention and done keeps every
interruption meaningful: either an agent needs you now, or it just finished.

> The author was explicit about signal quality: *"I don't want the icon blinking
> — no icons blinking. Also send push from the app on any attention / done (don't
> need on start)."*

## Acceptance criteria (EARS)
- When a task's status becomes attention, the system shall fire a native desktop
  notification.
- When a task's status becomes done, the system shall fire a native desktop
  notification.
- When a task's status becomes working (an agent starts), the system shall not
  fire a notification.
- The status icons shall be steady state indicators and shall never animate or
  blink to attract attention.

## Build notes
- Reuse the existing native notification path (E01) rather than a new mechanism;
  attention/done are just two more notification sources.
- Fire on transition *into* attention/done, not on every hook tick, so a
  persisting state doesn't re-notify.
- Pair with the idle-notification guard (H06) so a generic idle ping never
  masquerades as a real attention/done notification.
- "No blinking" is a UI constraint on H01's icons; keep them static.
