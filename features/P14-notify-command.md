---
id: P14
epic: P — Agent-facing control API
title: "External notifications — `taskman notify` adds a notification to the in-app panel; a notification id makes a new one replace the old"
size: S
requires: [P01, P06, P12, E04]
novel: true
---

## What
A non-gated control-API endpoint lets any external program or script put a
notification into the in-app notification panel (E04), next to the panel's own
derived notifications:

```
taskman notify --id backup-nightly --title "Backup failed" --text "rsync exit 23" --level error --task I565
taskman notify --id backup-nightly --title "Backup OK" --level ok          # replaces the one above
taskman notify --clear --id backup-nightly                                   # takes it back
taskman notify --title "Deploy finished" --url https://ci.example/run/42     # no id: always a new one
```

The **notification id** (`--id`, `notificationId` on the wire) is the caller's
handle for one notification: a new `notify` with the same id **replaces** the old
one — the old card disappears and the new one is added at the top with a fresh
time, even if the user had dismissed the old one. A script that reports the same
thing repeatedly (a nightly job, a polling check) therefore never stacks up
duplicates; it keeps exactly one card current. Without an id the app generates
one (printed by the CLI), so every call is a new notification.

`--clear --id X` removes notification X if it is in the panel — the caller's
equivalent of E04's auto-clear ("the condition is gone").

One optional action per notification: `--task I<N>` → **Open** (opens the
task), `--terminal <session>` → **Go to terminal**, `--url <http(s) url>` →
**Open link** (system browser). The icon follows `--level`
(info ℹ️ · ok ✅ · warn ⚠️ · error ❌) unless `--icon <emoji>` is given.

Wire format: `POST /api/v1/notify` with the usual `x-taskman-token` header and
`{"op":"add","notificationId":…,"title":…,"text":…,"level":…,"icon":…,"taskId":…,"terminal":…,"url":…,"source":{…}}`
or `{"op":"clear","notificationId":…}`. It answers `200 {"ok":true,"notificationId":…}`
at once — no request id, no Approve/Deny.

## Why it exists
E04 made the panel the one place that answers "what needs me", but only for
things the app can derive itself. Scripts, monitors, CI watchers and other
programs know about things the app can't see. P12's status bar and popup are for
a script's *own run* (working… → result); a notification is for a fact that
should wait in the inbox until handled, and that the script can update or
withdraw later. Without a stable id a repeating script would flood the panel.

## Acceptance criteria (EARS)
- The app shall accept `POST /api/v1/notify` on the control-API port, guarded by
  the per-boot token of P01; a missing or wrong token shall get `403` and change
  nothing.
- The app shall not put `notify` through the approval gate: a valid call shall
  take effect at once and be answered `200 {"ok":true,"notificationId":<id>}`.
- If `op` is `add` (the default) and `title` is empty, or `op` is `clear` and
  `notificationId` is empty, or `op` is anything else, then the app shall answer
  `400` with an error and change nothing.
- The app shall cap `title` at 200 characters, `text` at 2 000 and
  `notificationId` at 100; an unknown `level`, a malformed `taskId`, a non-http(s)
  `url` or a `terminal` with characters outside `[A-Za-z0-9_-]` shall be dropped
  rather than rejected.
- When an `add` arrives without a `notificationId`, the app shall generate a
  unique one and return it.
- When an `add` arrives, the app shall remove any notification with the same
  `notificationId` from the panel and add the new one at the top, with the
  current time — also when the user had dismissed the earlier one.
- When a `clear` arrives, the app shall remove the notification with that
  `notificationId` if present, and answer `200` either way.
- The panel shall show an external notification like E04's (icon, title, text as
  plain text, time added, ×), with one action button: **Open** for `taskId`,
  **Go to terminal** for `terminal`, **Open link** for `url` (in that order of
  preference when several are given).
- When the user clicks × on an external notification, the app shall remove it; it
  shall come back only through a new `add` with that id.
- External notifications shall never be removed by E04's reconciler (only by ×,
  Clear all, `clear` or a replacing `add`), and shall survive an app restart like
  the rest of the panel.
- The app shall raise no desktop notification and play no sound for `notify`.
- The app shall append each accepted call to the control-API audit log
  (`action:"notify"`) with op, notification id, title and the caller-supplied
  source.
- The `taskman` CLI shall provide `notify [--id <notificationId>] --title <t>
  [--text <s> | --text-file <path> | --text -] [--level info|ok|warn|error]
  [--icon <emoji>] [--task I<N>] [--terminal <session>] [--url <url>]` and
  `notify --clear --id <notificationId>`; it shall print the notification id,
  exit 0 once the app accepted the call and non-zero if the app is unreachable
  (scripts write `taskman notify … || true`).

## Build notes
- A third POST route beside `/api/v1/action` and `/api/v1/show`, same token
  check; forward the validated payload to the renderer over one IPC channel.
- Renderer: external items use kind `external` and keys `ext:<notificationId>`.
  E04's `reconcile` only touches the kinds it is told are loaded, so it never
  sees `external`; `upsertExternal` drops the old item **and** any dismissal of
  that key before adding, and dismissing an external item records no dismissal
  (there is nothing derived to suppress).
- A script outside an app terminal runs the CLI by path:
  `node <data root>/tasks-app/bin/taskman.mjs notify …` (P06).
