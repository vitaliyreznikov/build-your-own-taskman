---
id: P13
epic: P — Agent-facing control API
title: "Auto-approve rules — a machine-local allowlist lets named, trusted requests (e.g. \"open the transcripts task's terminal\") skip Approve/Deny"
size: S
requires: [P01, P02]
novel: true
---

## What
A machine-local, git-ignored file `.taskman-autoapprove.json` at the data root
lists requests the user has decided to trust once and for all:

```json
{
  "rules": [
    { "action": "open-terminal", "params": { "taskId": "I9" },
      "note": "FocusClock: a new meeting transcript is ready" }
  ]
}
```

A request that matches a rule is **approved automatically**: it skips the banner,
the desktop notification and the 10-minute expiry, and its executor runs at once.
Everything else about it stays the same — same verb, same validation, same
result, same `taskman status` polling, and an audit line in
`.api-requests.jsonl` that says it was auto-approved and by which rule.

A rule matches when its `action` equals the request's action and **every** key in
its `params` equals the request's param of the same name (exact, string
comparison). Params the rule does not mention are free: the rule above also
covers `taskman run I9 --prompt "…"` and `--new`, but not `taskman run I10`.

## Why it exists
P01 gates every agent request behind Approve/Deny, which is right for an agent
deciding on its own to start work. It is wrong for a small number of
**pre-agreed** hand-offs that fire unattended and always mean the same thing —
e.g. a meeting recorder that, after each call, asks to open the one standing
task that reads transcripts. There the prompt is pure friction: the user always
says yes, and when they are away the request expires and the work silently does
not happen. A per-machine allowlist keeps the gate for everything else, keeps
the trust decision with the user (a file only they edit, never an API call), and
keeps the audit trail.

## Acceptance criteria (EARS)
- The app shall read `.taskman-autoapprove.json` from the data root **on every
  incoming request** (no restart, no cache), and the file shall be git-ignored.
- If the file is missing, unreadable or not valid JSON, or `rules` is not an
  array, then the app shall treat it as having no rules (every request gated as
  in P01) and shall log the problem once per distinct error.
- A rule without a non-empty string `action` shall be ignored. A rule's `params`
  is optional; a rule with no `params` matches every request of that action.
- When a valid request (it passed the verb's `validate`) matches a rule, the app
  shall not show the approval banner or notification and shall not start the
  approval timeout; it shall run the verb's executor immediately, exactly as if
  the user had pressed Approve.
- When a request matches no rule, the app shall behave exactly as P01.
- The POST response shall be unchanged (`202 {requestId, status:"pending"}`), so
  callers need no changes; polling shall show `done`/`error` as usual.
- The audit log shall record an `auto-approved` event for the request, carrying
  the index and `note` of the matching rule, followed by its `done`/`error` event.
- There shall be no API to create, change or list rules: the file is edited only
  by hand.

## Build notes
- Match in `createRequest` (P01 spine), after `validate` and before surfacing.
  First matching rule wins; compare with `String(param) === String(ruleValue)`.
- Add the filename to `.gitignore` next to `.taskman-api.json`.
- Document the file next to the `taskman` CLI help ("auto-approve rules") so an
  agent that is asked to make a hand-off unattended knows the user can allow it.
