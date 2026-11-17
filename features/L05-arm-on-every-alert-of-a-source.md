---
id: L05
epic: L — PR-review blocking
title: Arm a task on every alert of a source (wildcard monitor reference)
size: M
requires: [L02, L03]
novel: true
---

## What
L02 arms a task on **one** monitor. L05 lets a task be armed on **every** alert
a source raises, so that one task acts as the inbox for a whole alerting system.
The reference is the source name followed by `*`, optionally narrowed by label
matchers on a Prometheus-compatible source:

```
| From | Type          | To                          |
|------|---------------|-----------------------------|
| I790 | runs-on-alert | vm:*                        |
| I790 | runs-on-alert | vm:*{env="production"}      |
| I790 | runs-on-alert | dd:*                        |
```

Everything else is L02: the same `runs-on-alert` relation, the same machine-local
`.monitor-sources.json`, the same "each check covers the gap since the last
successful one" window, the same K01 countdown, and the same L03 delivery into
the task's live terminal. What changes is the unit of an **incident**: for a
wildcard it is one **alert instance** (one alert rule plus its labels, or one
Datadog monitor plus its group) starting to fire, and one check can find many of
them across many rules.

| source `type` | wildcard id | an incident is |
|---|---|---|
| `prometheus` | `*` plus optional `{matchers}` | one `ALERTS` series (alertname + labels) entering `firing`, identified by its `ALERTS_FOR_STATE` activation time |
| `datadog` | `*` (optionally `*{tag:value}` as a monitor-event query) | one monitor alert event (monitor id + group) transitioning to `Triggered` / `Alert`, from the events API |
| `grafana`, `command` | — | not supported; the reference shows an error |

A check that finds new incidents **appends them to the task's inbox file for the day**
`<task dir>/alerts-YYYY-MM-DD.jsonl` (local date of the check) — one JSON object per incident, oldest first, each
naming the reference and source it came from — and then sends the task **one
short notice**: how many incidents, how many still firing, and the file and line
range just appended. The incidents live in the file, not in the prompt, so a
burst of hundreds costs one small message, nothing is truncated, and the file is
a durable history the agent can diff against. Fires on different sources
(`vm:*` and `dd:*`) land in the same inbox, each line saying which system raised
it, and reach the terminal as separate notices.

## Why it exists
Running two alerting systems side by side (a migration, a dual-run) raises one
question again and again: *did both fire?* Arming a task on each rule would mean
hundreds of `runs-on-alert` rows kept in sync with the rule set by hand. A
wildcard arms one task on the whole firehose of each system. The agent behind it
compares the two sides and hands every difference to the task that owns it.
TaskMan stays a delivery mechanism; the comparison is ordinary agent work.

## Acceptance criteria (EARS)
- The system shall accept `<source>:*` and, for a `prometheus` source,
  `<source>:*{<matchers>}` as monitor references in `runs-on-alert` rows, with
  the matchers validated as in L02.
- When a wildcard reference names a `grafana` or `command` source, the system
  shall show an `error` state for it naming the reason and shall not poll it.
- When a `prometheus` wildcard is checked, the system shall find every `ALERTS`
  series matching the matchers with `alertstate="firing"` at any point in the
  window, and shall treat each distinct (series labels, activation time) as one
  incident.
- When a `datadog` wildcard is checked, the system shall read the Datadog events
  API for monitor alert events in the window, and shall treat each
  (monitor id, group, event time) of a transition into the alert state, and
  of a transition into the warn state (a separate `warn`-level instance, the
  twin of a split warn-tier rule on the other stack), as one incident; a
  recovery shall close both tiers, and a move between alert and warn shall
  close the tier it leaves.
- Each check after the first shall also re-read the 10 minutes before its
  window, so that an event the source indexed late is still found; the
  handled set shall keep it from being delivered twice.
- The system shall react at most once per incident: it shall keep a record of
  the incidents already handled for each wildcard reference, so that a later
  check, an overlapping window or an app relaunch does not deliver the same
  incident again.
- When a wildcard reference is checked for the first time, the system shall
  record the alerts already firing as handled and shall not fire.
- When a check finds one or more new incidents and the task is not DONE or
  CLOSED, the system shall append every new incident to the day's inbox file
  `<task dir>/alerts-YYYY-MM-DD.jsonl` (local date of the check)
  (one JSON object per line, oldest first, with reference, source, alert name,
  labels or group, start time and whether it is still firing), and then make
  **one** L02 fire for that task whose prompt names the file and the line range
  appended.
- If appending to the inbox file fails, the system shall list up to 50 of the
  new incidents in the prompt instead and say how many were left out.
- Where a source sets `intervalSec` (≥ 60) in `.monitor-sources.json`, the
  system shall check that source's references at most once per that interval
  (any source type, not only `command`), with each check still covering the
  window since the previous successful one; "Check now" and a reference in
  `error` state shall not wait for the interval.
- Where a source sets `notifyEverySec` (≥ 60), the system shall still append
  every new incident to the inbox file at once, but shall send the task at most
  one notice per that interval for that reference; a notice shall cover every
  line appended since the previous notice (one or more file + line ranges), and
  incidents held back shall be sent once the interval has passed even if no new
  incident arrives.
- Where a source sets `quiet: true`, the system shall deliver its fires without
  the K01 countdown, the in-app countdown banner, a desktop notification or a
  sound; when such a delivery fails, the system shall show a notification
  without sound naming the task and the reason.
- The Monitors view shall show a wildcard row with the number of alerts firing
  on it now and the time of the last delivered incident.

## Build notes
- **Prometheus query.** Same range query as L02 without the `alertname`
  filter: `ALERTS_FOR_STATE{<m>} and ignoring(alertstate) ALERTS{alertstate="firing",<m>}`.
  Key each incident by `alertname` + sorted labels + activation time. The number
  of series can be large, so step up the query resolution sooner than L02 does.
- **Datadog events.** `POST https://api.<site>/api/v2/events/search` with
  `filter.query = "source:alert"` (plus any narrowing from the reference),
  `filter.from` / `filter.to` = the window, and paging through `meta.page.after`.
  Keep events whose alert transition is `Triggered` (or `Re-Triggered`, or a
  status of `error` / `alert`). Read the monitor id and group from the event
  attributes, and use the monitor name for the prompt. This sees incidents that
  opened and closed inside the gap, which L02's per-monitor current-state check
  cannot.
- **Handled set, not a watermark.** L02's single `handledThrough` timestamp is
  not enough when many alerts start in the same second. Keep a set of handled
  incident keys per reference in `.monitor-status.json`, and prune keys older
  than the 30-day window cap.
- **Inbox file.** Append-only (`appendFile`), never rewritten by the app; count
  the existing lines first to report the range. The agent owns it after that —
  it may rotate or trim it once handled. One file per local day keeps each file small
  and makes "what fired yesterday" one file; several wildcard references armed
  on one task share the day's file.
- **Firehose cadence.** A wildcard is heavier than one rule, so give its source
  its own entry with `intervalSec` (e.g. 120) instead of slowing the shared
  60 s poll; a second source entry may point at the same URL.
- **Throttle and quiet are per source.** A firehose wakes an agent every check;
  `notifyEverySec` (e.g. 600) batches notices while the inbox stays complete,
  and `quiet` keeps a routine stream from popping up on the desktop. Keep the
  held-back line ranges and the last notice time per (reference, task) in
  `.monitor-status.json` so a restart neither loses nor repeats them.
- **Status entry.** For a wildcard, `state` is `firing` while any alert matches
  and `labels` carry `count=<n>` instead of one series' labels.
