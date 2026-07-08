---
id: H05
epic: H — LLM / agent status via Claude hooks
title: Marker fallbacks
size: S
requires: [H04]
novel: false
---

## What
When an agent's message carries no explicit status marker, the system still
produces a useful status by falling back in order: first to the **last narration
line** of the message (a human-readable summary of what the agent just said),
then, if there is nothing usable, to a **mechanical tool label** (e.g. the name
of the last tool the agent ran).

## Why it exists
The marker protocol (H04) depends on the agent cooperating. Agents don't always
emit a marker — so the status must degrade gracefully rather than go blank or
stale. A best-effort narration line keeps the hover message informative; the tool
label guarantees there is always *something* truthful to show, so the supervision
signal never silently dies.

> The status contract is meant to keep the human oriented at all times — *"end
> every turn's final message with one marker line."* When the agent skips it, the
> app still has to answer "what is this agent doing?" — hence the fallbacks.

## Acceptance criteria (EARS)
- When an agent message contains no status marker, the system shall use the last
  narration line of the message as the status message.
- When no usable narration line exists, the system shall fall back to a
  mechanical label derived from the last tool the agent used.
- The system shall apply fallbacks only after marker detection (H04) finds no
  marker.
- The Stop hook shall use the final message's first line as the done summary when
  a done marker is missing (rather than showing an empty done state).

## Build notes
- Fallback order is strict: explicit marker (H04) → last narration line → tool
  label. Implement it as a single resolver so the card/tab UI never sees a null
  status while an agent is active.
- The tool-label fallback is intentionally mechanical — it is the floor, not a
  nice summary; its job is to never leave the supervisor blind.
- Coordinate the done-summary fallback with the Stop-hook logic in H04 so a
  markerless stop reads as an informative done, not a blank one.
