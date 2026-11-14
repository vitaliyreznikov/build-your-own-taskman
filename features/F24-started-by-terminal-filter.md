---
id: F24
epic: F — In-app terminals
title: "\"Started by\" terminal filter — tell PR-review auto-launched terminals apart from the rest"
size: S
requires: [F20, F22, L01]
novel: true
---

## What
Every terminal remembers **what started it**. Today one origin is distinguished:

- **`pr-review`** — the app opened it by itself because a PR blocker got a human
  review (L01 / O07 "PR reviewed, no live terminal" reaction).
- everything else — opened by the user, autorun, a `taskman run` worker, a report
  link, a reboot restore — has no origin tag and counts as **"Me & others"**.

The sidebar's **Terminals** block gets a **Started by** control under the category
checkboxes: **All · 🔁 PR review · Me & others**. It narrows the tab strip together
with the category filter (F20) — e.g. `bolt` + `🔁 PR review` = only the bolt terminals
that PR reviews spun up. A tab started by a PR review carries a small `🔁` mark.

## Why it exists
PR-review reactions open terminals on their own, often several in a burst after a
reviewer pass. Mixed into the strip with the sessions the user is actively driving,
they're hard to triage: "which of these did *I* open, and which are the bot-launched
review follow-ups I should sweep through?" A one-click split answers it without
closing anything.

## Acceptance criteria (EARS)
- When the app opens a terminal because of a PR-review reaction (task or subgoal),
  it shall record that terminal's origin as `pr-review`.
- When a PR-review reaction focuses an already-open terminal, the app shall not change
  that terminal's origin.
- The origin shall survive an app restart (stored machine-locally per session name)
  and shall be forgotten when the terminal is closed.
- While **Started by = 🔁 PR review**, the tab strip shall show only terminals with
  origin `pr-review` that also pass the category filter; while **Me & others**, only
  terminals without it; **All** shows both.
- The F22 ★ Starred union still applies: a starred terminal is shown in ★ mode
  regardless of Started by.
- The sidebar shall show the count of terminals per Started-by option, and the chosen
  option shall persist (local UI preference).
- A tab with origin `pr-review` shall show a `🔁` mark with a tooltip
  "started by a PR review".
- Terminals that were already open before this feature have no origin and count as
  **Me & others**.

## Build notes
- Store origins as `{sessionName: "pr-review"}` in localStorage next to the F22 starred
  set; prune entries when the terminal is removed.
- Pass `origin: 'pr-review'` from the PR-review open path into
  `openTaskTerminal` / `openSubgoalTerminal`; only a *new* tab records it.
- Extend the single strip-visibility predicate (F22 `terminalVisible`) so the strip,
  the attention cycle and active-tab re-homing keep agreeing.
- Room to grow: `alert` (L02 monitor fire) is the natural next origin; the control
  becomes a list of origins with "Me & others" as the remainder.
