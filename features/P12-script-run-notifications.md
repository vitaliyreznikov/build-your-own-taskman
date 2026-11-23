---
id: P12
epic: P — Agent-facing control API
title: "Script run notifications — a local script tells the app it started (closable status bar at the bottom) and finished (centered popup that stays until closed)"
size: M
requires: [P01, P06, E01]
novel: true
---

## What
A new, **non-gated** control-API endpoint lets any local script report the life of one
run to the app:

```
taskman notify start  --id disk-guard --title "Disk cleanup" --text "free 19 GB < 20 GB, cleaning caches…"
taskman notify finish --id disk-guard --title "Disk cleanup: +12.9 GB" --text-file I565/summary.txt \
                      --level ok --task I565
```

- **start** → a slim **status bar at the bottom** of the window: spinner, title, text and
  how long ago it started, with a **×** to close it. Several runs stack.
- **finish** → the bar for that `id` disappears and a **popup in the center** of the
  window shows the title and the full result text. It stays until the user closes it.
  Several finished runs queue up and show one at a time ("1 of 3").
- `--task I565` adds an **Open I565** button to both, which selects that task.

Wire format: `POST /api/v1/notify` with the usual `x-taskman-token` header and
`{"phase":"start"|"finish","id":…,"title":…,"text":…,"level":"info"|"ok"|"warn"|"error","taskId":…}`.
It answers `200 {"ok":true}` straight away: there is no request id, no Approve/Deny and
nothing to poll.

## Why it exists
Housekeeping scripts run unattended: a monitor's command check (L04) that frees disk
space, a nightly sync, a backup. Their results end up in a log file that nobody opens,
so you only learn they ran (or failed) by going to look. Approval-gated verbs (P01) are
wrong for this: a script can't wait for a human just to say "I'm done", and a status
message changes nothing, so there is nothing to approve. A start bar the user can ignore
and a finish popup that has to be closed fit how much attention each phase deserves.
"Started" is context; "finished, here is what changed" is something the user should
actually read.

## Acceptance criteria (EARS)
- The app shall accept `POST /api/v1/notify` on the control-API port, guarded by the same
  per-boot token as P01; a missing or wrong token shall get `403` and show nothing.
- The app shall not put `notify` through the approval gate: a valid call shall be shown at
  once and answered `200 {"ok":true}`.
- If `phase` is not `start`/`finish`, or `id` or `title` is empty, then the app shall answer
  `400` with an error and show nothing. `title` shall be capped at 200 characters and
  `text` at 20 000. Anything else (an unknown `level`, a malformed `taskId`) shall be
  ignored rather than rejected.
- When a `start` arrives, the app shall show a status bar at the bottom of the window with
  the title, the text and the time since it started, plus a close control. A second `start`
  with the same `id` shall replace that bar, not add another one.
- When the user closes a start bar, the app shall hide it; a later `finish` for that `id`
  shall still show its popup.
- When a `finish` arrives, the app shall remove the start bar with the same `id` (if any)
  and show a modal popup centered in the window with the title, the level (as a color/icon)
  and the full text, whitespace preserved.
- The finish popup shall stay until the user closes it, with its Close button or Esc;
  clicking outside it shall not close it, and it shall never close on a timer.
- While more than one finish popup is waiting, the app shall show them one at a time, oldest
  first, with a position indicator ("1 of 3").
- When `taskId` names an existing task, the bar and the popup shall offer a button that
  selects that task; the popup's button shall also close the popup.
- When a `finish` arrives while the app window is not focused, the app shall also fire a
  native desktop notification with the title, so the waiting popup is noticed.
- The app shall render `title` and `text` as plain text, never as HTML.
- The app shall append each accepted call to the control-API audit log (`action:"notify"`)
  with phase, id, title and the caller-supplied source.
- The `taskman` CLI shall provide `notify start|finish --id --title [--text <s> | --text-file
  <path> | --text -] [--level] [--task]`; `--text -` reads stdin. It shall exit 0 once the
  app accepted the call and non-zero if the app is unreachable, so a script can write
  `taskman notify … || true` and keep running when the app is closed.
- Notifications shall not survive an app restart; nothing is written to the data root.

## Build notes
- Keep it off the action registry: it is a second POST route beside `/api/v1/action`,
  behind the same token check, that forwards `{phase,id,…}` to the renderer over one IPC
  channel. The renderer store keeps `runs: Map<id, start>` and `finished: queue`.
- Plain-text rendering (`white-space: pre-wrap`, monospace) is enough for a summary table
  and closes the injection hole a script-supplied HTML string would open.
- A script launched outside an app terminal has no `taskman` on its PATH: call
  `node <data root>/tasks-app/bin/taskman.mjs notify …` (P06). The CLI reads the endpoint
  from the discovery file, so it works from cron, launchd or a monitor check alike.
- Reasonable first users: an L04 command check that cleans up before an agent is woken
  (start when it begins cleaning, finish with "freed X GB, where"), backup/sync scripts.
