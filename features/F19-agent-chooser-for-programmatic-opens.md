---
id: F19
epic: F — In-app terminals
title: Agent chooser for programmatic opens (API wait + scheduled 30s countdown)
size: M
requires: [F15, F17, P02, K01, L01]
novel: true
---

## What
Extend the F15/F17 agent chooser (Claude vs Devin + model tier) to the opens that
today happen **without a human click**, so the LLM is a deliberate choice there
too — while never silently dropping scheduled work. Two shapes, both **only when
over the weekly budget** (healthy balance stays silent, exactly as today):

- **API `open-terminal`** (a task worker asks the app to open a tab): the chooser
  **replaces** the generic Approve/Deny banner for this one verb. Picking an agent
  *is* the approval and launches it with the chosen model; **Cancel denies** the
  request. It **blocks** — there is no countdown; the request waits for the human
  (bounded only by the control API's existing approval timeout).
- **Scheduled autorun (K01/K02) and PR-review auto-resume (L01/O07)**: the chooser
  appears with a **30-second countdown** and a pre-selected default. If the human
  picks within 30 s, that wins; if they don't (timeout, Esc, dismiss), the system
  proceeds with **exactly today's `unattendedAgent()` default** (Devin when over
  budget, honoring any caller hint) — the work never stalls waiting on a human who
  isn't there.

The existing **manual** over-budget chooser (F15) is unchanged: no countdown, it
waits for a pick, and Cancel drops the open.

## Why it exists

> The manual "open terminal" button already lets me pick Claude or Devin when I'm
> over budget. But a worker opening a tab through the API, and the scheduler/PR
> poller opening one on their own, still pick silently — the most expensive opens
> (unattended, repeated) are the ones I can't steer. I want to choose there too.
> For an API open a person is right there asking, so wait for my pick. For a
> scheduled fire nobody may be watching, so give me 30 seconds and then just go.

The budget gate is deliberate: when quota is healthy there's nothing to decide, so
these flows keep launching with no interruption. The chooser only appears when the
choice actually matters (over budget), and the countdown makes the unattended case
safe — a fire at 3am still runs.

## Acceptance criteria (EARS)
- When an `open-terminal` API request is received **and** the weekly balance is
  negative, the system shall present the agent chooser in place of the Approve/Deny
  banner; selecting an agent shall approve the request and launch that agent with
  the selected model, and Cancel shall deny the request.
- When an `open-terminal` API request is received and the weekly balance is **not**
  negative, the system shall show the normal Approve/Deny banner and launch as it
  does today (no chooser).
- When a scheduled autorun or PR-review auto-resume fires **and** the weekly balance
  is negative, the system shall present the chooser with a 30-second countdown and a
  pre-selected default; a pick within the window shall be used, and on timeout or
  dismissal the system shall launch the current `unattendedAgent()` default.
- When a scheduled autorun or PR-review auto-resume fires and the balance is **not**
  negative, the system shall launch silently as it does today (no chooser).
- The manual over-budget chooser shall keep its current behavior: no countdown,
  waits for a pick, Cancel opens nothing.
- No programmatic open shall be lost because a chooser went unanswered: the API
  waits (approval-timeout bounded), and the unattended countdown always resolves to
  a launch.

## Build notes
- **Generalize `AgentChoice`**: add `mode: 'manual' | 'api' | 'unattended'`, a
  highlighted `defaultAgent`, and an optional countdown `deadline` (epoch ms). The
  modal renders a ticking "auto-selecting <default> in Ns" only when `deadline` is
  set, highlights the default button, and swaps copy/Cancel-label per mode
  (`Deny` for `api`). Add generic `setAgentChoice`/`clearAgentChoice` store actions
  so the non-component flows (ipc) can drive the one slot.
- **Authoritative timeout lives in the caller, not the modal**: the unattended
  helper owns the 30s `setTimeout` that resolves the default and clears the slot;
  the modal's countdown is display-only. Guard the clear by identity (only clear if
  the current `agentChoice` is still this one) so overlapping choosers don't wipe
  each other — single-slot, latest-shown-wins, others resolve to their default.
- **Unattended helper** (`chooseUnattended(label, hint) → {agent, model}`): off/flag
  or healthy → resolve the `unattendedAgent()` result immediately (silent). Over
  budget → show the countdown chooser; resolve on pick or on the 30s default. Wire
  it into the autorun-fire and PR-review-open subscribers, threading the returned
  model into `openTaskTerminal`/`openSubgoalTerminal` (F17 already accepts it).
- **API path**: in the control-API request subscriber, when `action ===
  'open-terminal'` and over budget, show the chooser instead of pushing the banner
  row; a pick stashes `{agent, model}` keyed by `requestId` and approves via the
  existing resolve IPC, Cancel denies. The exec subscriber reads the stashed pick
  (falling back to `unattendedAgent(hint)`) and passes the model through. Leave
  `create-task`/`rebind` and healthy-budget opens on the normal banner.
- **Gate**: reuse the F15 `OVER_BUDGET_AGENT_CHOICE` flag and the same
  `getOverBudget()` signal; fail open to a silent Claude launch on any usage error
  (never block a terminal on a budget check).
