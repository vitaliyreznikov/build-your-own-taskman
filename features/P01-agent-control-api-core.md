---
id: P01
epic: P — Agent-facing control API
title: Agent-facing control API core (loopback HTTP + command registry + approval gate + `taskman` CLI)
size: L
requires: [A02, E01, F09]
novel: true
---

## What
A single, extensible **control API** that lets a process — most often an LLM agent
running inside a TaskMan-spawned terminal — ask the app to *do* things, each one
gated by an explicit user approval. This feature is the reusable **spine**; the
individual verbs (the first is P02, "open a terminal") are separate features that
plug into it. It is deliberately built so that adding the Nth command later costs
one registry entry, not a new endpoint.

Three layers:

### 1. Transport — a loopback HTTP endpoint
The app exposes a token-guarded HTTP surface bound to loopback only, following the
exact security posture of the existing chat-ingest server (127.0.0.1, a shared
`X-Taskman-Token` header, Node `http`, no external interface). Two routes under a
versioned prefix:

- `POST /api/v1/action` — body `{ action, params, source }` where `source` is
  `{ taskId?, termId? }` identifying the caller. The server validates the token,
  looks the `action` up in the command registry, validates `params`, creates a
  **pending request** with a freshly minted `requestId`, and returns **`202
  { requestId }` immediately**. It does *not* wait for the work or the approval —
  this is a fire-and-forget contract.
- `GET /api/v1/action/:requestId` — returns the request's current state:
  `{ status, result?, error? }` where `status ∈ pending | approved | denied |
  done | error`. This is the poll path a caller uses if it wants the outcome.

On boot the app writes the live port + token to a well-known file in the data root
(e.g. `.taskman-api.json`, gitignored) so a local caller can discover how to reach
it without hard-coding anything.

### 2. Core — command registry + approval gate + audit
- **Command registry.** A map `action → { validate(params) , describe(request) ,
  execute(request) → result }`. `validate` rejects malformed params at
  `POST` time (before a pending request is ever created, so the caller gets a
  synchronous `400`). `describe` produces the human sentence shown in the approval
  UI. `execute` performs the action when approved and returns the JSON result.
  **Registering a new verb is the whole cost of adding an API call.** P01 ships the
  registry with zero or one verbs; P02 registers the first real one.
- **Approval gate.** A newly created pending request is pushed to the renderer over
  a new `api:request` channel; the renderer shows an in-app banner **and** a
  desktop notification (E01) carrying the `describe` sentence and **Approve /
  Deny** controls. On **Approve**, main runs `execute`; on success the request
  becomes `done` with the result, on throw it becomes `error`. On **Deny** it
  becomes `denied`. A request left unanswered past a timeout becomes `denied`
  ("expired"). The gate is written to consult a **per-action approval policy** so a
  future feature can allowlist specific verbs to auto-approve; in P01 the policy is
  "everything requires approval," with no exceptions.
- **Audit log.** Every request and its terminal outcome is appended to a
  gitignored, append-only JSONL side file in the data root
  (`.api-requests.jsonl`): one line at creation, one line at resolution. This is
  the durable backing that `GET /api/v1/action/:requestId` reads, and the history
  the user can inspect. It is append-only and never rewritten.

Pending requests live in memory keyed by `requestId`; the JSONL is the durable
record. A request that is still pending when the app quits is not resurrected —
the caller's poll will simply stop finding progress, and a fresh request can be
made. (Persisting live pending requests across restarts is a later feature if it
proves needed.)

### 3. Surface — the `taskman` CLI
A small CLI is the thing an agent (or a human, or a script) actually invokes; it is
transport-agnostic sugar over the HTTP core and is what makes the API *usable by an
LLM* — a self-documenting command beats teaching curl-with-a-token in a prompt.

- It reads the port + token from the boot-written discovery file, and defaults the
  `source` fields from the `TASKS_TASK_ID` / `TASKS_TERM_ID` environment variables
  that every task terminal already exports (F02/F05) — so an agent doesn't have to
  know its own ids.
- `taskman <verb> [args…]` → `POST`s the action, prints the `requestId`, exits 0.
  (Fire-and-forget: it does **not** block on approval.)
- `taskman status <requestId>` → `GET`s and prints `{ status, result?, error? }`.
- `taskman help` / `--help` → lists the registered verbs and their arguments.
- If the endpoint is unreachable (app not running), it exits non-zero with a clear
  message; there is no offline queue in v1.

P01 defines the CLI's dispatch, discovery, and the `status`/`help` verbs; the
action verbs themselves come from their own features (P02 onward).

## Why it exists
Everything an external process can do to TaskMan today is *file-poking* — write
`tasks.md` and hope the watcher reacts (that is how autorun is triggered, K01).
That is fine for "arm a schedule," but it cannot *ask the app to open a terminal*,
cannot carry a return value, and has no approval step. As TaskMan grows into the
place agents both read from and act through, it needs a real, first-class control
surface: a named verb, validated params, an explicit human yes/no, a result, and an
audit trail. Building the spine once — registry + approval + audit + CLI — means the
long tail of future verbs ("update this field," "add a blocker," "schedule a run,"
"post a note") is each a few lines, and every one of them inherits the same
approval gate and audit log for free. The approval gate is the non-negotiable part:
an agent can *request* anything, but nothing happens to the user's board or machine
without the user saying yes.

## Acceptance criteria (EARS)
- The system shall expose an HTTP endpoint bound to loopback only, guarded by a
  shared token header, following the existing chat-ingest server's security model.
- The system shall write the live API port and token to a gitignored discovery file
  in the data root on startup.
- When a well-formed, authenticated `POST /api/v1/action` names a registered action
  with valid params, the system shall create a pending request, return its
  `requestId`, and respond without waiting for approval or execution.
- When a `POST /api/v1/action` fails token check, names an unregistered action, or
  carries params the action rejects, the system shall respond with an error status
  and create no pending request.
- When a pending request is created, the system shall surface it to the user as an
  in-app banner and a desktop notification carrying the action's human description
  and Approve/Deny controls.
- When the user approves a pending request, the system shall run the action's
  executor and record the request as `done` with its result, or as `error` if the
  executor throws.
- When the user denies a pending request, or the request times out unanswered, the
  system shall record it as `denied` and shall not run the executor.
- The system shall append one JSONL line to the audit file when a request is created
  and one when it resolves, and shall never rewrite existing lines.
- When a client `GET`s `/api/v1/action/:requestId`, the system shall return that
  request's current status and, once resolved, its result or error.
- The `taskman` CLI shall discover the endpoint from the discovery file, default the
  caller `source` from `TASKS_TASK_ID`/`TASKS_TERM_ID`, submit the action, print the
  returned `requestId`, and exit without blocking on approval.
- The `taskman status` command shall print the current status/result/error of a
  given `requestId`, and `taskman help` shall list the registered verbs.
- When the endpoint is unreachable, the CLI shall exit non-zero with a clear error
  and shall not silently queue the request.
- The approval gate shall consult a per-action approval policy; in this feature the
  policy shall require approval for every action.

## Build notes
- Reuse the chat-ingest server as the template for the listener (loopback bind,
  shared-token guard, Node `http`, start on app init / stop on quit). It can be a
  sibling route set on the same server or a second small server; either way keep the
  token + loopback discipline identical.
- The registry, approval gate, and audit belong in the **main** process. Executors
  run in main; the only renderer involvement is the approval UI (`api:request` push
  + an Approve/Deny reply channel) and, where a verb needs a renderer-only action,
  the executor delegating to the renderer exactly as autorun's fire does
  (main → renderer message → existing store call). P02's open-terminal is such a
  case.
- Keep `validate` synchronous and cheap so a bad `POST` fails fast with a `400`;
  keep `execute` async and allowed to fail (its throw becomes `error`).
- The discovery file and the audit file are transient/local: add both to the app's
  gitignore list alongside `.llm-status.json` and the doc-index cache. The audit
  file is append-only — open in append mode, one JSON object per line, never a
  read-modify-write.
- The `source.taskId`/`source.termId` are advisory (they feed the approval
  sentence and the audit line); do not trust them for authorization — the loopback
  bind + token are the trust boundary.
- Ship the CLI as a tiny standalone script placed on `PATH` (or invoked via a stable
  absolute path from the seeded terminal prompts). Because task terminals already
  export the ids, the CLI needs no per-terminal configuration.
- Seed a one-line mention of `taskman help` into the terminals' work-on-task prompt
  so agents discover the surface — the same channel F12/F13 already use to teach the
  agent its environment.
