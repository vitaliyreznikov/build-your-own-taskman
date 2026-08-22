---
id: P06
epic: P — Agent-facing control API
title: "Control CLI always reachable — `taskman` on PATH in app terminals + unconditional, zero-install invocation"
size: S
requires: [P01, P02]
novel: false
---

## What
A **reachability** fix, not a new capability. The control CLI (`taskman`, P01) and its
verbs (`run`, P02/P04; `create-task`, P03; `rebind`, P05) all work — but an agent
running in an app terminal can't actually *call* them, and the instructions tell it to
give up. Two things are wrong:

1. **`taskman` is never on PATH.** P01 documented installation as a manual
   `ln -s …/bin/taskman.mjs /usr/local/bin/taskman`, but that directory is
   root-owned (needs `sudo`) so the symlink was never created. `which taskman` fails in
   every terminal the app spawns.

2. **Every instruction is gated on that false condition.** The line typed into each
   task terminal, the AGENTS.md control-API section, and the `run-task` skill all say
   *"If the `taskman` CLI is on your PATH, …"*. Since it never is, an LLM runs
   `which taskman` / `taskman help`, both fail, and it stops — never discovering that
   the CLI is a dependency-free script it can run directly. The guard meant to keep the
   hint from misfiring instead makes it **never fire**.

P06 makes the CLI genuinely reachable and rewords the instructions so an agent cannot
get stuck:

1. **Put `taskman` on PATH for every app-spawned terminal.** The terminal launcher
   already prepends `/opt/homebrew/bin` to the child `PATH`; it additionally prepends
   the app's own `bin/` directory, and that directory ships an **extensionless
   executable `taskman`** (a tiny wrapper that execs `taskman.mjs` with node). No
   `sudo`, no mutation of the user's machine, no global install — the command resolves
   by name inside any terminal the app opens.

2. **Make the invocation unconditional, with a guaranteed-working fallback.** The
   terminal prompt line, AGENTS.md, and the `run-task` skill drop the *"if on PATH"*
   framing and instead state: run `taskman <verb>`; if it isn't found, run
   `node <repo>/tasks-app/bin/taskman.mjs <verb>` — which always works (the script has
   no dependencies and self-discovers endpoint + identity). The absolute-path form is
   documented as the reliable fallback so an agent in any environment has a path that
   cannot fail the `which` check.

Nothing about the request/approve/execute flow, params, wire `action`, or result shape
changes.

## Why it exists
The control API's value is zero if the agents it's built for can't reach it. The P01–P05
mechanism is correct, but a manual, `sudo`-requiring install step that never ran left the
CLI unreachable, and instructions phrased as a precondition on that missing install turned
"unreachable" into "the agent concludes it must not use this." The fix removes both the
precondition (ship the CLI on the terminal's PATH) and the trap (tell the agent an
invocation that always works), so "ask the app to run another task" becomes a thing an
agent actually does rather than a capability it reasons its way out of.

## Acceptance criteria (EARS)
- When the app spawns a task terminal, the child process `PATH` shall include the app's
  `bin/` directory, so that `taskman` resolves by name without any user install.
- The app's `bin/` directory shall contain an executable named `taskman` (no extension)
  that invokes the existing `taskman.mjs` with the same arguments and exit code.
- The terminal work-on-task / work-on-subgoal prompt shall describe reaching the control
  API without a conditional on the CLI being installed, and shall name the
  `node …/taskman.mjs <verb>` fallback as an invocation that always works.
- AGENTS.md's control-API section shall present `taskman <verb>` as available in app
  terminals and document the `node …/taskman.mjs` fallback; it shall not present being
  on PATH as a precondition the reader must satisfy.
- The `run-task` skill shall likewise give the unconditional invocation plus fallback.
- Running `taskman.mjs` directly (by absolute path, with `node`) shall continue to work
  with no arguments beyond the verb — discovering endpoint and caller identity from the
  environment as today.
- No change shall be made to the server-side action registry, params schema, executor,
  approval flow, or result shape.

## Build notes
- Launcher (`electron/terminal/ptyManager.ts`): the spawned `env.PATH` gains the
  resolved `tasks-app/bin` path ahead of the inherited PATH, next to the existing
  `/opt/homebrew/bin` prepend. Resolve the dir relative to the app root, not `__dirname`
  of the compiled bundle, so it points at the real `bin/` in dev and packaged builds.
- Wrapper `bin/taskman`: a POSIX `#!/bin/sh` one-liner
  `exec node "$(dirname "$0")/taskman.mjs" "$@"` (committed executable, `chmod +x`).
  Keeping the wrapper separate leaves `taskman.mjs` importable/uninstalled-runnable as
  before; the extensionless name is what PATH resolution needs.
- Prompt (`electron/terminal/prompts.ts`): rewrite the `CONTROL_API` constant to the
  unconditional form + fallback. This text is shared by the work-on-task and
  work-on-subgoal prompts; the create-task prompt does not include it and is unchanged.
- AGENTS.md: update the "Agent control API (`taskman`)" bullet that currently reads
  "Needs `taskman` on your PATH (`ln -s …`)" — replace with "available on PATH in app
  terminals; elsewhere run `node …/tasks-app/bin/taskman.mjs`".
- `run-task` skill (user env `~/.claude/skills/run-task/`, not the app repo): same
  reword; keep it self-contained so it works even read outside a terminal.
- No behavioural change to the CLI's argument parsing or the HTTP contract — this is
  purely reachability + copy.
