---
id: F17
epic: F — In-app terminals
title: Model-tier dropdown in the Claude/Devin chooser
size: S
requires: [F15]
novel: true
---

## What
The F15 over-budget chooser gains a **model-tier dropdown** above its two launch
buttons. The user picks a *tier* once; the tier is provider-agnostic — each row
carries **both** a Devin model id and a Claude model alias — and the button they
click (**Devin** or **Claude**) decides which of the two is sent as `--model` on
the launch command.

The four tiers (single dropdown, in this order):

| Tier      | Dropdown label   | Devin `--model`         | Claude `--model` |
| --------- | ---------------- | ----------------------- | ---------------- |
| `default` | `default`        | *(none — Devin default)* | *(none — Claude default)* |
| `luna`    | `luna — sonnet`  | `gpt-5-6-luna-medium`   | `sonnet`         |
| `terra`   | `terra — opus`   | `gpt-5-6-terra-medium`  | `opus`           |
| `sol`     | `sol — fable`    | `gpt-5-6-sol-medium`    | `fable`          |

`default` selects neither flag — each agent starts on its own configured default
(the pre-F17 behaviour). The dropdown defaults to `default`, so a user who ignores
it and just clicks a button gets exactly today's launch.

The tier→model mapping lives in **one** config table so the strings are trivial to
retune as vendor model names churn. The label text (`luna — sonnet`) is derived
from that table, not hard-coded in the modal.

Scope note: the dropdown lives **only** inside the F15 chooser (the modal that
already appears when the weekly balance is negative). Healthy-balance opens still
launch Claude directly with no dialog and no `--model` (F12 unchanged), and the
**unattended** path (silent Devin) launches Devin with **no** `--model` (its own
default) — there is no human to pick a tier.

## Why it exists

> The author already gets a Claude-vs-Devin prompt when over budget. But "Devin"
> and "Claude" each span several models at very different price/capability points
> — Devin's cheap Luna vs a Claude Opus. Choosing the *agent* without choosing the
> *model* leaves the most consequential cost/quality lever on the floor. One
> dropdown, cross-mapped so the same tier means "cheapest / mid / top" on either
> vendor, lets the per-terminal decision that's already being made also set the
> model.

The tiers are cross-mapped on purpose: the user thinks "how much do I want to
spend on this terminal?" (cheap `luna`, mid `terra`, premium `sol`/`opus`) and the
agent button is a separate, orthogonal choice. A single ordered dropdown beats two
provider-specific ones because the decision is one axis, not two.

## Acceptance criteria (EARS)
- When the over-budget chooser is shown, the system shall render a model-tier
  dropdown offering `default`, `luna`, `terra`, `sol` (labelled with their Claude
  equivalents), defaulted to `default`.
- When the user selects a tier and clicks **Claude**, the system shall launch
  Claude with that tier's Claude `--model` (or no `--model` for `default`).
- When the user selects a tier and clicks **Devin**, the system shall launch Devin
  with that tier's Devin `--model` (or no `--model` for `default`).
- When the user leaves the dropdown at `default` and clicks either button, the
  system shall launch that agent exactly as it does today, with no `--model` flag.
- When a terminal is opened on the **unattended** path (autorun / PR-resume), the
  system shall launch Devin with no `--model` flag and shall present no dropdown
  (unchanged F15 unattended behaviour).
- When the chooser is cancelled, the system shall launch nothing (unchanged).

## Build notes
- **Single source of truth**: a `MODEL_TIERS` table (ordered list of
  `{ id, label, claude?, devin? }`) in the terminal config module. The modal maps
  it to `<option>`s; the launch layer looks up the row by `id` and reads the
  provider column. `default` has neither `claude` nor `devin` set → emit no flag.
- **Thread a `model` tier id** (not the resolved flag) from the chooser through the
  same plumbing F15's `agent` already rides: `AgentChoice.choose(agent, model)` →
  `openTaskTerminal`/`ensureTaskTerminal` opts → the tab object →
  `TerminalOpenOptions` → `openSession` → `autoStartAgent`. Resolve the tier id to
  the concrete `--model` string **at the command-build step** (`autoStartAgent`),
  where the agent is already known, so the mapping stays in one place.
- **Command build**: append `--model <resolved>` to both the Claude launch (`claude
  --model <x>`, typed before the prompt) and the Devin launch
  (`devin --model <x> -- <prompt>`). Omit the flag entirely when the tier resolves
  to nothing for that agent (i.e. `default`).
- **Claude aliases** `opus` / `sonnet` / `fable` are accepted by `claude --model`
  directly; **Devin** has no short aliases for the GPT-5.6 families, so use the
  concrete `gpt-5-6-<family>-medium` ids (medium thinking) — guaranteed-valid model
  strings from `devin models list`.
- **Styling**: mirror the existing native `<select>` used for task priority; add a
  labelled row inside the chooser between the copy and the action buttons.
- **Backward compat**: everything defaults such that not touching the dropdown, and
  every non-chooser launch path, behaves identically to pre-F17.
