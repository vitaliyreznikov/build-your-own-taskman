---
id: H04
epic: H — LLM / agent status via Claude hooks
title: Status markers in answer text
size: M
requires: [H03]
novel: false
---

## What
The agent embeds status directly in its answer text as marker lines —
`🤖 working: …`, `⏳ waiting: …`, `⚠️ attention: …`, `✅ done: …`. The hooks scan the
**whole message** for these markers, and when several appear, **last match wins**
(the most recent phase is the current one). The text after the marker becomes the
hover message.

## Why it exists
This is the author's core invention for supervising agents at scale: a status
contract every session carries, compiled down to a one-character pictogram the
app can read. It lets the agent narrate its phase in the same prose it's already
writing, and lets the board extract a machine-readable state from it — no
separate reporting channel to forget.

> The protocol, in the author's words: *"Report status inside your answers: when
> your phase changes add a line '🤖 working: <what you're doing>', and end every
> turn's final message with one marker line — '✅ done', '⏳ waiting: <external
> process …>' or '⚠️ attention: <anything you need from me>' (a question =
> attention, not done)."*

And on scanning robustly — the bug that forced whole-message scanning:

> *"LLM answer contains '⏳ waiting' but not in the beginning of the message — this
> is why it missed … fix this by reading all the message for pictograms."*

## Acceptance criteria (EARS)
- The system shall recognize the four marker forms (working / waiting /
  attention / done) anywhere in the agent's message, not only at the start.
- When multiple markers appear in one message, the system shall use the last
  matching marker as the current status.
- The system shall use the text following the marker as the status's hover
  message.
- The Stop hook shall use a `done`/`attention`/`waiting` marker to set the final
  status, and shall not misread an attention or waiting marker as done.
- When the Stop hook's input carries the final message text, the Stop hook shall
  classify from that text and finish well within its hook timeout, never waiting
  on the transcript file.
- When no marker is present, the system shall apply the fallbacks defined in H05.

## Build notes
- Scan the entire message body for pictograms (`transcript-tail.mjs`); do not
  anchor the match to line start — this was a real miss the author reported.
- "Last match wins" models phase progression within a turn: an agent may say
  `working:` then end with `done:`; the ending marker is the truth.
- Stop-hook: prefer an explicit marker; if the marker is `attention`/`waiting`,
  keep that state — do not coerce a stopped turn into `done` just because the
  turn ended. (The done-summary fallback lives in H05.)
- Read the final message from the Stop payload's `last_assistant_message`, not
  from the transcript. Claude Code ≥ 2.1.296 appends the final text to the
  transcript only *after* Stop hooks return, so a hook that polls the transcript
  for it never sees it and is killed at its timeout — every turn then stays
  `working` forever. Keep the transcript read only as a fallback for older
  agents that don't send the field, and cap its wait below the hook timeout.
- Keep the marker vocabulary exactly aligned with the four card icons (H01) and
  the waiting-vs-attention semantics (H10).
