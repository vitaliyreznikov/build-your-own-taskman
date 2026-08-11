---
id: P04
epic: P — Agent-facing control API
title: "`run` verb + \"task worker\" vs \"subagent\" — naming so \"run I396\" reaches the API, not a subagent"
size: S
requires: [P02]
novel: false
---

## What
A **naming** change, not a new capability. The control API already lets one session
ask the app to open an independent session on another task (P02, `open-terminal`).
But the word *agent* collides: a coding LLM has its own **subagent** primitive (an
in-context helper it spawns and awaits), so when the user says *"run I396"* the LLM
reasonably reaches for that subagent tool instead of the app's API — doing I396's
work inside its own context on the wrong task, invisible to the board.

P04 removes the collision with three cheap, purely-additive moves:

1. **Primary verb `run`.** `taskman run <taskId> [--goal <subgoalId>] [--agent claude|devin]`
   becomes the human-facing verb, aliasing P02's `open-terminal` (the wire
   `action` stays `"open-terminal"` — server registry unchanged; only the CLI adds
   the alias). `open-terminal` keeps working as a documented low-level alias.

2. **A noun: "task worker".** The thing `run` launches is named a **task worker** —
   an *independent peer session* on another task (its own terminal, board card,
   `TASKS_TASK_ID`, git commits, user-approved). It is explicitly contrasted with a
   **subagent** (the LLM's in-context helper: shares the current task, returns a
   result, then disappears, no terminal/card/commits). The CLI help and the P01
   approval banner are reworded to this vocabulary (*"…wants to run I396 as its own
   task worker"*).

3. **A skill + an AGENTS.md block** that make an LLM pick the API path. AGENTS.md
   gains a "Two ways to get more work done — subagent vs task worker" section. A
   broad **`run-task` skill** (triggering on *run / launch / kick off / hand off*
   other work — deliberately **not** keyed on a task-ID pattern) routes such
   requests to `taskman run` rather than a subagent, the same way the `schedule`
   skill routes scheduling to autorun.

Nothing about the request/approve/execute flow, params, or result changes.

## Why it exists
The API's whole point (P02) is that work belonging to *another task* should run as
its own first-class session the human can see and steer — not be absorbed into the
requesting LLM's context. That intent survives only if the LLM actually chooses the
API when the user expresses it. "run I396" is the natural phrasing, and the default
LLM reading of "run"/"agent" points at its own subagent machinery. The mechanism was
right; the vocabulary was doing the opposite of its job. Naming the verb `run`, the
result a **task worker**, and adding a skill whose one-line description is the
disambiguation surface makes the correct path the obvious one — with no behavioural
risk, since the wire contract is untouched.

## Acceptance criteria (EARS)
- The CLI shall accept `run` as a verb equivalent to `open-terminal`, submitting the
  same `action = "open-terminal"` request with the same params.
- The CLI shall continue to accept `open-terminal` unchanged (documented as an alias).
- The CLI help shall present `run` as the primary verb and describe the launched
  session as a **task worker**.
- The P01 approval description for this action shall use the "run … as its own task
  worker" wording.
- AGENTS.md shall document the distinction between a **subagent** (in-context helper)
  and a **task worker** (independent approved peer session) and when to use each.
- A `run-task` skill shall exist whose description triggers on requests to run/launch/
  hand off other work (without depending on a task-ID pattern) and directs the LLM to
  `taskman run` instead of a subagent.
- No change shall be made to the server-side action registry, params schema, executor,
  or result shape.

## Build notes
- CLI: fold `run` into the existing `open-terminal` branch (`verb === 'open-terminal'
  || verb === 'run'`); keep the emitted `action` string as `"open-terminal"`.
- Banner: reword P02's `describe()` to *"`<caller>` wants to run `<taskId>` … as its
  own task worker"*; this is copy only.
- The skill mirrors `schedule`'s shape (a `SKILL.md` with a broad `description:` that
  is the real routing signal) and lives in the user's `~/.claude/skills`, not the app
  repo — it is environment config, like `schedule`.
- Keep the skill description free of `I<N>` / "task ID" phrasing so it fires on the
  intent ("run the onboarding work as its own worker") not just the shorthand.
