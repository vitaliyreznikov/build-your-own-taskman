---
id: H01
epic: H — LLM / agent status via Claude hooks
title: Live status icons on cards
size: M
requires: [B02]
novel: false
---

## What
Each task card carries a small live status icon reflecting what its agent is
doing right now: 🤖 working, ⏳ waiting, ⚠️ attention, or ✅ done. Hovering the
icon reveals the exact message the agent last reported (e.g. "attention: which
migration order do you want?"). The icon is driven by a status signal the agent
updates, not by anything the user sets on the card.

## Why it exists
The whole product is an attention-routing machine. When one human supervises
many parallel agents, the bottleneck is not writing code — it is knowing, at a
glance across N cards, which agent needs you. The status icon is that glance: it
turns the board into a live dashboard of what every agent is doing.

> The author's original spec: *"I want to see LLM status as icon on the task
> pile. It could be In progress (LLM works) / Attention Needed (means my
> attention). When I move mouse the exact message from LLM should be shown. We
> need to provide LLM API to update such status."*

## Acceptance criteria (EARS)
- When a task has a current agent status, the system shall render the
  corresponding icon (working / waiting / attention / done) on its card.
- When the user hovers the icon, the system shall show the exact status message
  the agent last reported for that task.
- When a task has no status, the system shall render no status icon (a clean
  card, not a placeholder).
- When the underlying status changes, the system shall update the card icon
  without the user reloading the board.

## Build notes
- Keep this a pure render of a status signal owned elsewhere: the card reads the
  current status/message per task and maps it to an icon + tooltip. Do not let
  the card compute or set status.
- Icon vocabulary is exactly four states — working, waiting, attention, done —
  matching the marker protocol (H04) and the waiting-vs-attention semantics
  (H10). Do not add intermediate states.
- The status source is the transient status file (H02), written by the hooks
  (H03); this feature is only the consumer/UI. A manual CLI (`llm-status.mjs`)
  can set status by hand for testing and human use.
- Respect "no icons blinking" (see H07): the icon is a steady state indicator,
  never an animated attention-grab.
