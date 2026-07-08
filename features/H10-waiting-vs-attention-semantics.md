---
id: H10
epic: H — LLM / agent status via Claude hooks
title: waiting-vs-attention semantics
size: S
requires: [H04]
novel: false
---

## What
Two superficially similar states are kept strictly distinct. ⏳ **waiting** means
the agent is blocked on an *external* process — a job, CI, a review — where no
action from the user is required. ⚠️ **attention** means the agent needs *the
user*. An agent may be waiting on the world, but it must never be silently
waiting on the human. Accordingly, the "needs you" surface includes **attention
only**, never waiting.

## Why it exists
This is the human-in-the-loop philosophy compiled down to a one-character
distinction. The entire point of the status system is routing the user's own
attention — so the line between "the world is busy" and "you are needed" has to be
exact. Blur it, and either the user is pestered by external waits or, worse, an
agent sits blocked on them unnoticed.

> The author iterated on this precisely: *"'waiting' means external waiting —
> some operations, external review, etc. Everything for me — should be
> attention."* And bluntly: *"I mean 'never waiting from me'."* When the "needs
> you" function wrongly included waiting, he flagged it: *"'needs you' function
> includes 'waiting' — this is wrong."*

## Acceptance criteria (EARS)
- The system shall treat waiting as blocked on an external process with no user
  action required.
- The system shall treat attention as requiring the user.
- The system shall never represent "blocked on the user" as waiting.
- The "needs you" surfaces (counts, jump-to-next, notifications) shall include
  attention only and shall exclude waiting.

## Build notes
- Enforce the semantics at the marker layer (H04): the `⏳ waiting:` marker is for
  external waits; anything the agent needs from the human must resolve to
  attention.
- Make every "needs you" consumer attention-only: the jump-to-next-attention nav
  (H09), the sidebar terminal-state counts, and the attention/done notifications
  (H07). This was fixed across several commits — treat it as an invariant, not a
  per-surface toggle.
- Encode the contract in the session's launch prompt so agents emit `waiting`
  only for genuinely external blocks — never as a stand-in for needing the user.
