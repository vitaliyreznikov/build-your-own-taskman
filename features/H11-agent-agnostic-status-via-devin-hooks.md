---
id: H11
epic: H — LLM / agent status via Claude hooks
title: Agent-agnostic status via Devin CLI hooks and exports
size: L
requires: [H03, H04, H05, H06, H08, F15]
novel: false
---

## What
Devin CLI terminals participate in the same per-terminal LLM status protocol as
Claude terminals. Devin lifecycle hooks write the transient `.llm-status.json`
entry keyed by `TASKS_TERM_ID`, and Devin's per-terminal ATIF export is parsed for
the same status markers (`🤖 working:`, `⏳ waiting:`, `⚠️ attention:`, and
`✅ done:`). Explicit markers are authoritative; lifecycle and tool fallbacks keep
the status useful when a marker is absent.

## Why it exists
F15 makes Devin a first-class unattended and over-budget alternative, but a
terminal that cannot report its state is invisible to the supervision cockpit.
The status contract must follow the agent, not the vendor: the same card badges,
tab icons, attention navigation, and waiting semantics should work regardless of
whether Claude or Devin is running.

## Acceptance criteria (EARS)
- When Devin runs inside a TaskMan terminal, its lifecycle hooks shall update the
  status entry keyed by `TASKS_TERM_ID` for prompt submission, tool activity,
  stopping, permission requests, and session end.
- When Devin's export contains one or more status markers, the system shall use
  the last marker in the current turn and its trailing text as the status message.
- When an export has no marker, the system shall fall back to a useful narration
  or tool label, without leaving Devin's status blank.
- When Devin requests a permission or user decision, the system shall report
  `attention`, and external CI/review/deploy waits shall remain `waiting`.
- When Devin exits, its status entry shall be cleared.
- Claude's existing hook behavior shall remain unchanged.
- A Devin terminal shall show its live status in task cards, task terminal tabs,
  the full-screen terminal tab strip, and attention navigation exactly as a
  Claude terminal does.

## Build notes
- Launch Devin with a unique per-terminal `--export` path and pass the export
  location to the hook through the terminal environment.
- Reuse the existing concurrency-safe status-file writer and marker parser; add a
  Devin adapter for the documented hook stdin payloads rather than pretending
  Devin has Claude's `transcript_path`.
- Devin command hooks receive event JSON on stdin and support
  `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PermissionRequest`, `Stop`,
  `SessionStart`, and `SessionEnd`. The hook must never block Devin or emit
  non-JSON stdout that could be interpreted as hook control output.
- Parse the export after it is written, retrying briefly at stop/session boundaries
  because export persistence may lag the lifecycle event. A lifecycle fallback is
  required when no export is available.
- Keep this feature local and machine-only: status files and exports must not be
  git-synced or committed.
