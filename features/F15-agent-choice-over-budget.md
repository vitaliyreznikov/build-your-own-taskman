---
id: F15
epic: F — In-app terminals
title: Agent choice (Claude vs Devin) when over weekly budget
size: M
requires: [F12, J01]
novel: true
---

## What
When the Claude account is **over its weekly budget**, the app stops silently
launching Claude in every new task terminal and instead offers — or picks — a
second coding agent, **Devin**. "Over budget" reuses the pace signal J01 already
computes: the 1-week window's `elapsed% − used%` is negative (you're burning quota
faster than the clock; the sidebar already renders this red as `−X%`).

Two entry points diverge:

- **Manual open** (the user clicks "open terminal" on a task or subgoal): a small
  in-app chooser appears — *"Weekly balance is −X%. Use Claude or Devin for this
  terminal?"* — with **Claude** / **Devin** / cancel. The chosen agent is launched.
- **Autostart** (a scheduled autorun fire (K01), or a PR-review auto-resume (L01)):
  no human is present, so the app **silently uses Devin** — no dialog.

When the weekly balance is healthy, nothing changes: Claude launches directly with
no dialog on both paths (F12 behaviour, untouched).

Devin is a drop-in terminal agent. Its CLI (`https://docs.devin.ai/cli`) is an
interactive REPL launched, with a preloaded prompt, as:

```
devin -- <the same work-on-task prompt F12 would give Claude>
```

So "use Devin" reuses the entire F12 typed-into-tmux mechanism — only the launch
command and its readiness handling change. The work-on-task prompt (and the
status-marker contract inside it) is byte-identical to Claude's.

## Why it exists

> The author hit `−10%` on the Claude weekly goal but keeps a Devin.ai account for
> exactly this: when Claude is rationed, unattended work should keep flowing on the
> other agent, and attended work should let the human pick per terminal.

Auto-start (F12) assumes one agent with unlimited-enough quota. Once you're over
the weekly budget, blindly spending more Claude quota — especially on *unattended*
autorun/PR-resume fires that the human never sees — is the wrong default. This
feature makes the budget state (J01) actually steer which agent runs, keeping the
human in the loop only when they're actually there to answer.

## Acceptance criteria (EARS)
- When a task/subgoal terminal is opened **manually** and the weekly balance is
  negative, the system shall present a chooser offering Claude or Devin (and a
  cancel), and shall launch the chosen agent with the work-on-task prompt.
- When a terminal is opened by an **unattended** trigger (scheduled autorun or a
  PR-review auto-resume) and the weekly balance is negative, the system shall
  launch **Devin** with the work-on-task prompt and shall present no chooser.
- When the weekly balance is **not** negative, the system shall launch Claude
  directly with no chooser, on both the manual and unattended paths (unchanged
  F12 behaviour).
- When Devin is chosen/selected, the system shall start it as
  `devin -- <work-on-task prompt>`, reusing the same prompt text (including the
  status-marker contract) that Claude would receive.
- When the over-budget signal cannot be determined (usage fetch fails), the system
  shall fail **open** to Claude (never block a terminal on a usage error).
- When the whole behaviour is disabled by config, the system shall always launch
  Claude and present no chooser regardless of budget.

## Build notes
- **Over-budget signal**: compute in the main process from the same data J01's
  widget uses (`fetchClaudeUsage()` → `sevenDay` window; pace delta =
  `elapsed% − used%`). Expose it over IPC so both the autorun/PR path (main) and
  the chooser modal (renderer) read one source of truth. Threshold is a tunable
  constant (default `0`; can be raised to e.g. `−10%` or "quota exhausted").
- **Launch path**: generalize F12's "type `claude`, poll ready, type prompt" to an
  agent parameter. For Devin the prompt rides in the launch command
  (`devin -- <prompt>`) as a single submission, so no separate ready-poll/type-
  prompt step is needed. Keep Claude's path exactly as-is.
- **Prompt reuse**: use the identical work-on-task / work-on-subgoal builders so
  the status-marker contract is preserved verbatim across agents.
- **Prerequisite**: the Devin CLI must be installed and authed
  (`brew install --cask devin-cli`; `devin auth login`). If it's absent at runtime
  the pane simply shows `command not found` / a login prompt — visible, not silent.
- **Known limitation**: the H03/H04 live-status hooks are Claude-Code-specific and
  do not fire for Devin, so a Devin terminal's card icon stays generic even though
  Devin is *asked* (via the shared prompt) to emit the same markers. Making the
  supervision layer agent-agnostic is out of scope here.
- Gate the whole feature behind a boolean config (reference app:
  `OVER_BUDGET_AGENT_CHOICE`) so it can be turned off, mirroring F12's flag.
