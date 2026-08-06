---
id: H12
epic: H — LLM / agent status via Claude hooks
title: Dedicated permission-approval status icon
size: S
requires: [H06, H08, H11]
novel: false
---

## What
Represent an agent waiting for the human to approve a command or other
permission request as a distinct `approval` status, rather than the generic
`attention` status. Render a lock icon (`🔐`) for approval waits everywhere
terminal status is shown, while retaining `attention` for questions and other
user decisions.

## Why it exists
A permission prompt is urgent for the user, but it is different from an agent
asking a question: the user must authorize an operation and should be able to
recognize that situation at a glance. A dedicated icon makes approval prompts
immediately distinguishable without weakening the existing attention routing.

## Acceptance criteria (EARS)
- When Claude or Devin emits a permission-request lifecycle event, the
  per-terminal status shall become `approval` without requiring a marker in the
  agent's text.
- When an agent asks a question or exits plan mode, the status shall remain
  `attention`, not `approval`.
- When the status is `approval`, task cards, task badges, terminal tabs, the
  terminal-state tally, subgoal terminals, and approval/attention navigation
  shall render the `🔐` icon and label it as approval required.
- The `approval` status shall remain part of the needs-user priority and shall
  be included wherever `attention` is surfaced for user action.
- When approval is granted or the agent continues, subsequent lifecycle or
  marker status updates shall replace `approval` normally.
- Existing Claude and Devin status behavior for working, waiting, attention,
  and done shall remain unchanged.

## Build notes
- Extend the shared `LlmState` union and status-file validation with
  `approval`.
- Map `approval` to `🔐` and `Approval required` in the shared status icon and
  label tables.
- Update status ordering and needs-user predicates so approval is at least as
  urgent as attention; do not treat it as external `waiting`.
- Change the Claude notification/permission hook and Devin `PermissionRequest`
  hook to write `approval`; leave `AskUserQuestion` and `ExitPlanMode` as
  `attention`.
- Update the status tests and run the app test suite plus renderer/main builds.
