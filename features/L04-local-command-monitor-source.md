---
id: L04
epic: L — PR-review blocking
title: Local command checks as a monitor source (e.g. low disk space)
size: S
requires: [L02]
novel: false
---

## What
L02 arms a task on a monitor that lives in a remote alerting system. Some of the
things worth waking an agent for are only visible **on this machine** — free disk
space, a local service that stopped answering, a stuck sync directory — and have
no alert rule anywhere. L04 adds a fourth source type to `.monitor-sources.json`,
**`command`**: a named set of shell checks the app runs itself.

```json
{
  "sources": {
    "disk": {
      "type": "command",
      "intervalSec": 900,
      "checks": {
        "free-lt-20g": "test $(df -k /System/Volumes/Data | awk 'NR==2{print $4}') -lt 20971520"
      }
    }
  }
}
```

The monitor reference is `<source>:<check>` — here `disk:free-lt-20g` — in an
ordinary L02 `runs-on-alert` row:

```
| I565 | runs-on-alert | disk:free-lt-20g |
```

Everything after the check — once-per-incident, baseline on first sight, the K01
countdown, L03 delivery into a live terminal, the timeline line, the Monitors
view — is L02 unchanged.

### A check's result is its exit status
Each check is run with `/bin/sh -c` and a timeout:

| exit | meaning |
|---|---|
| `0` | **firing** (the condition holds — "free space is below 20 GB") |
| `1` | ok |
| anything else, killed, or timed out | the check failed → the reference shows `error` |

Writing the check as a `test` makes the convention natural (`test` exits 0 when
true, 1 when false, 2 on a syntax error). The command's output is never stored or
logged — the error state carries only a short reason ("check exited 2", "check
timed out after 10s").

### Its own cadence
The L02 poll tick stays at 60 s. A `command` source may set `intervalSec`
(≥ 60; default 60 = every tick); its checks then run only when that long has
passed since the reference's **last successful** check. A failed check does not
count, so it is retried on the next tick. **Check now** (and adding a row) runs
every command check regardless of its interval.

### Current state only — the incident start lives in the status cache
A local check answers "is it true now", not "when did it start" or "what
happened while I wasn't looking", so — like L02's Datadog source — it has no
catch-up window. The app keeps the bookkeeping itself:

- An **ok → firing** transition is a new incident whose start is the time of the
  first firing check. The start is persisted with the reference's status
  (`firingSinceMs`), so it survives a relaunch.
- **firing → firing** is the same incident (same start), so a check that stays
  firing does **not** re-run the task every interval.
- An ok check ends the incident; the next firing check is a new one and fires
  again.
- A failed check in the middle of a streak leaves it alone (still the same
  incident when the check recovers firing).
- First sight follows L02: a check already firing when first seen is a baseline,
  not a fire.

## Why it exists
The motivating case: a runbook task ("free up disk space on this Mac") that only
needs to run when the disk is actually getting full. A calendar cadence runs it
too often or too late; a remote monitoring stack doesn't watch a laptop. A shell
one-liner already expresses the condition precisely, and L02 already knows how to
turn "a condition became true" into exactly one agent run with the human holding
the countdown — the only missing piece is letting the condition be a local
command.

## Acceptance criteria (EARS)
- The system shall accept a source of `type` `command` in `.monitor-sources.json`
  whose `checks` map a check name to a shell command, and a monitor reference
  `<source>:<check>` naming one of them.
- When a `command` reference is evaluated, the system shall run its command via
  `/bin/sh -c` with a timeout and treat exit status 0 as firing and 1 as ok.
- When the command exits with any other status, is killed, or times out, the
  system shall show the reference in the `error` state with a short reason, and
  shall not log or store the command's output.
- When the reference names a check that the source does not define, the system
  shall show the reference in the `error` state naming the missing check.
- Where a `command` source sets `intervalSec`, the system shall evaluate its
  references only when at least that many seconds have passed since the
  reference's last successful check, and shall otherwise keep its last status
  unchanged; a failed check shall be retried on the next poll tick.
- When the user presses Check now or adds a `runs-on-alert` row, the system shall
  evaluate every `command` reference regardless of its interval.
- When a `command` check fires after its previous successful check was ok, the
  system shall treat it as a new incident starting at that check's time and
  react to it once (L02).
- While a `command` check keeps firing on consecutive successful checks, the
  system shall treat it as the same incident and shall not react again, also
  across an app relaunch and across intervening failed checks.
- When a `command` reference is checked for the first time and is already
  firing, the system shall record it as handled and shall not react (L02
  baseline).
- The Monitors view and its add form shall accept `command` references like any
  other.

## Build notes
- **Pure helpers, shared:** put the rules in the module that already parses
  references, so they are unit-testable without the app — "is this check due"
  (last-ok time, interval, now, force), "exit status → firing/ok/error", and
  "(firing now, previous streak start, now) → incident list + new streak start".
  Feed the incident list into L02's existing once-per-incident rule (fire for
  incidents starting after the newest handled one): a streak's start never
  changes, so it is handled exactly once.
- **Not due ≠ checked.** A skipped reference keeps its previous status entry
  as-is (including `lastOkMs`), otherwise the interval would restart on every
  tick.
- **Environment:** run with the same PATH augmentation as the credential
  commands (Homebrew, `/usr/sbin`, `/sbin`) — a GUI-launched app has a minimal
  PATH, and `df`/`diskutil` live in the sbin directories on some systems.
- **Timeout:** default 10 s, configurable per source (`timeoutSec`, capped). A
  check that hangs must not stall the rest of the poll for long.
- **The window display:** a command check covers "now" only, so show its window
  as the check time, not "previous check → now".
- **The armed task's column still matters:** L02 skips DONE/CLOSED tasks, so a
  runbook task armed this way must stay open (e.g. NEXT) to be run.
- **Not in scope:** stdout-valued checks with thresholds in the config (write the
  comparison in the command), and a `command` check with an incident history.
