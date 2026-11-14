---
id: L02
epic: L — PR-review blocking
title: Run a task when a monitor (by ID) fires
size: M
requires: [L01, K01, D04, L03]
novel: false
---

## What
A task can be **armed on an alerting monitor**: when that monitor starts firing,
the app launches the task's agent terminal — the same cancelable K01 countdown,
the same F12 auto-start — with an extra prompt that names the alert. It is L01's
"wait on an external thing, wake the agent when it happens" applied to a monitor
instead of a pull request.

The trigger is a `runs-on-alert` row in the git-synced `relations.md` (D04 —
just another relation type), where `To` is a **monitor reference**:

```
| From | Type          | To                                         |
|------|---------------|--------------------------------------------|
| I665 | runs-on-alert | vm:DialTcpIssues{env="production"}         |
| I700 | runs-on-alert | grafana:vmsingle-readonly-14d              |
| I701 | runs-on-alert | dd:12345678                                |
```

A reference is `<source>:<id>`. `<source>` names an entry in a **machine-local
source list**; `<id>` is whatever that kind of source calls a monitor:

| source `type` | `<id>` is | "firing" means |
|---|---|---|
| `prometheus` — any Prometheus-compatible query API (Prometheus, VictoriaMetrics, a Grafana datasource proxy in front of either) | an alert rule name, optionally followed by PromQL label matchers `{k="v",…}` | the `ALERTS{alertname="<id>",alertstate="firing",…}` series exists (Prometheus and vmalert both write it) |
| `grafana` — Grafana-managed alerting | the alert rule **UID** | the rule's state is `firing` (`alerting`) |
| `datadog` (optional) | the numeric monitor ID | `overall_state` is `Alert` |

The source list lives in `.monitor-sources.json` next to the data (gitignored —
it holds hostnames and the way to fetch a credential, never the credential):

```json
{
  "sources": {
    "vm":      { "type": "prometheus", "url": "https://grafana.example.com/api/datasources/proxy/uid/victoriametrics",
                 "tokenCmd": "security find-generic-password -s grafana.example.com -w" },
    "grafana": { "type": "grafana", "url": "https://grafana.example.com",
                 "tokenCmd": "security find-generic-password -s grafana.example.com -w" },
    "dd":      { "type": "datadog", "site": "datadoghq.com",
                 "apiKeyCmd": "…", "appKeyCmd": "…" }
  }
}
```

`tokenCmd` is a shell command whose stdout is the bearer token (a keychain
lookup, a password-manager CLI); the app runs it when it needs the token and
never writes the result anywhere.

### Firing = an incident; every check covers the whole gap since the last one
The app polls every armed reference on a short interval (default **60 s**) for
as long as it is running — **focused or not**. Unlike L01's review poll this is
not focus-gated: the reason to arm on a monitor is usually that the incident
happens when nobody is looking (a 03:00 brownout whose state must be captured
while it lasts).

A reference's **incident** is identified by when it started firing (the alert's
`activeAt`). The app reacts on the **edge** — a new incident — never on the
level, so a monitor that keeps firing for an hour launches one agent, not sixty.
The row stays armed until a human (or an agent) deletes it.

**A check is a window, not a snapshot.** Each check asks the source for every
incident that **fired at any point since the previous *successful* check of that
reference** — not only "is it firing right now". The app may have been closed
overnight, the laptop asleep for a weekend, or the source unreachable for an
hour: the next successful check covers that whole gap, so an incident that
started *and resolved* while nobody was polling is still found. A failed check
does not advance the window. The window is capped at **30 days**; a longer gap is
truncated and the prompt says so.

All new incidents found in one check produce **one** run (not one per incident),
and the prompt lists them — each with its start time and whether it is still
firing — because an agent that wakes after a night with eight brownouts should
see all eight and look at the pattern, not be launched eight times.

**Baseline on first sight:** a reference checked for the first time has no
previous check, so its window is just "now": an incident already firing then is
recorded as handled, not fired — arming means "the *next* time it fires". (The
Monitors view shows it as firing, so you can still run the task by hand.)

### The reaction
For each armed task not in DONE/CLOSED:

1. Enqueue it on the **K01 autorun queue** with the usual ~15 s cancelable
   countdown (notification + in-app banner). Several monitors firing at once
   launch one terminal at a time, exactly like overdue autoruns.
2. If the task already has a live terminal → **deliver the same prompt into that
   terminal's running agent session** (L03) instead of starting a second agent;
   only if delivery fails, fall back to a desktop notification ("🚨
   DialTcpIssues firing — I665").
3. Otherwise open the task's terminal (F12 + F09 IN PROGRESS) and append an
   extra prompt in the same submission:

   > Monitor `vm:DialTcpIssues{env="production"}` fired since the last check
   > (2026-11-14 01:10 → 07:42): incidents started 02:13 (resolved), 04:12
   > (resolved), 07:31 (still firing); labels: env=production, severity=page, ….
   > This task is armed to run when it fires: if it is still firing, do the
   > time-sensitive part now; then look at what happened and report. Ask me for
   > approval before changing anything.

4. Append a `{"kind":"monitor", …}` line to the task's `timeline.jsonl` if it is a
   v2 task (O05), naming the reference and the incident start.

Cancelling the countdown skips **this incident** only; the row stays armed.

### State and UI
- **`.monitor-status.json`** (machine-local, gitignored, like `.pr-status.json`):
  per reference — `state` (`firing` | `ok` | `error`), current `activeAt`,
  labels of the firing series, `checkedAt`, `handledActiveAt` (the last incident
  already reacted to), and `error` text. `handledActiveAt` is what makes an app
  relaunch mid-incident not fire twice.
- A separate **Monitors view** (its own sidebar entry, next to PRs — not a
  section of the PRs view): every `runs-on-alert` row with its task, reference,
  live state badge (🚨 firing since … / ✅ ok / ⚠️ error), the last incident
  reacted to, and the window the last successful check covered; an **add**
  control (task + reference), **remove** per row, and **Check now**. A reference
  whose source name is not in `.monitor-sources.json` shows as an error naming
  the missing source.
- The status entry also keeps `lastOkAt` — the end of the last successful
  check's window, i.e. where the next window starts.

## Why it exists
L01 made one external wait — a PR review — visible and self-resuming. The other
recurring wait in ops work is **"run this when the thing happens again"**:
capture DNS/conntrack state during the next brownout, grab a heap profile the
next time memory alerts, re-check a fix the next time the page fires. Without a
trigger the human has to be awake, notice the page, remember which task wanted
it, and open it — and the window has usually closed by then. Arming the task on
the monitor's own ID makes the alert itself start the investigation, with the
human still holding the off switch (the countdown) and the approval (the
prompt).

## Acceptance criteria (EARS)
- The system shall let a task be armed on one or more monitor references,
  persisted as `runs-on-alert` rows (from = task, to = `<source>:<id>`) in the
  git-synced `relations.md`.
- The system shall read monitor sources from a machine-local, gitignored
  `.monitor-sources.json`, supporting at least the `prometheus` and `grafana`
  source types; `datadog` is optional.
- The system shall obtain a source credential only by running that source's
  configured command, and shall not persist the credential.
- While the app is running, the system shall poll every armed reference on a
  fixed interval (default 60 s) whether or not the window is focused, and shall
  poll immediately when a row is added and on "Check now".
- Each check of a reference shall cover the window from the end of its previous
  **successful** check to now (capped at 30 days), so that incidents which
  started and resolved while the app was closed, asleep, or failing to reach the
  source are still found; a failed check shall not advance the window.
- When a `prometheus` reference is checked, the system shall find every incident
  in the window in which an `ALERTS` series with that `alertname`,
  `alertstate="firing"` and every given label matcher existed, identified by its
  `ALERTS_FOR_STATE` activation time.
- When a `grafana` reference is checked, the system shall find every transition
  into the firing state in the window from the rule's state history, plus the
  current incident if the rule is firing now.
- When a check finds one or more incidents newer than the last one handled and
  the task is not DONE or CLOSED, the system shall begin **one** K01 countdown for
  that task whose prompt lists every new incident with its start time and
  whether it is still firing.
- The system shall react at most **once per incident**: a later check, or an app
  relaunch, shall not fire again for an incident already handled.
- When a reference is checked for the first time, the system shall treat an
  incident already firing as handled and shall not fire.
- When the countdown elapses and the task has no live terminal, the system shall
  open its terminal (F12, F09) with the monitor prompt appended to the
  work-on-task prompt in one submission.
- When the countdown elapses and the task has a live terminal, the system shall
  deliver the monitor prompt into that terminal's agent session (L03) and shall
  not start a second agent; when delivery fails, it shall show a notification
  naming the monitor and the task.
- When the user cancels the countdown, the system shall skip that incident and
  shall keep the row armed.
- When the task is a v2 task, the system shall append one `monitor` line to its
  timeline per fired incident.
- When a reference cannot be evaluated (unknown source, credential command
  fails, HTTP or parse error), the system shall show an `error` state for it and
  keep polling the others.
- The system shall list every armed reference with its task, state and
  last-checked time in a **Monitors view of its own** (a sidebar entry separate
  from the PRs view), with add, remove and Check now.

## Build notes
- **One more poller, modelled on L01's.** Same shape as the PR poller: read the
  rows, group by reference (one HTTP call per reference even when several tasks
  arm on it), compare with the cached status, react, write the cache, push it to
  the renderer. Different cadence (60 s) and **no focus gate**.
- **Prometheus check is one range query over the window.**
  `GET <url>/api/v1/query_range` with
  `ALERTS_FOR_STATE{alertname="X",<m>} and ignoring(alertstate) ALERTS{alertname="X",alertstate="firing",<m>}`,
  `start` = previous successful check, `end` = now, `step` 60 s (grown so the
  range stays under ~10 000 points). Every distinct sample value is one incident
  (the value of `ALERTS_FOR_STATE` is the activation unix time; the `and` drops
  the pending phase). An incident is still firing when its series has a sample
  at the last step. Parse the reference as `name` + optional `{…}` and splice the
  matchers in verbatim after validating them against
  `^[a-zA-Z_][a-zA-Z0-9_]*(=|!=|=~|!~)"[^"]*"$`.
- **Grafana check is the state history plus the rules list.**
  `GET <url>/api/v1/rules/history?ruleUID=<uid>&from=<ms>&to=<ms>` returns the
  rule's transitions (`time`, `prev`, `next`); each `next` starting `Alerting` is
  an incident start. `GET <url>/api/prometheus/grafana/api/v1/rules` (one call per
  source per tick) says whether it is firing now and its current `activeAt`.
- **Datadog check** (if built): `GET https://api.<site>/api/v1/monitor/<id>`
  with `DD-API-KEY` / `DD-APPLICATION-KEY`; `overall_state === "Alert"`; incident
  start = `overall_state_modified`. This is the current state only — a Datadog
  incident that opened and closed inside the gap is missed unless the monitor
  events API is also read; say so in the source's docs.
- **Incident bookkeeping.** Keep `handledThrough` — the start time of the newest
  incident already reacted to — and fire for incidents that start after it. With
  window checks this is enough: an older incident cannot appear in a later
  window.
- **Reuse the autorun queue, don't build a second countdown.** Add an "external
  fire" entry point to the K01 scheduler (a transient target carrying an
  `extraPrompt`), so the countdown, one-at-a-time queue, post-fire gate and the
  live-terminal rule are shared. The extra prompt rides the same path L01 uses
  (`extraPrompt` appended in the F12 submission). An external target patches
  nothing on cancel or fire — the row is the arm, and it stays.
- **Credentials:** run `tokenCmd` with a timeout, cache the token in memory for a
  few minutes, drop it on a 401/403 and re-run once.
- **New relation type** = D04: add `runs-on-alert` to the relation-type union and
  to the reader's known-types set, or a later write of `relations.md` drops the
  rows.
- **Not in scope:** subgoal-level arming (a `monitor` blocker kind in a v2
  document), and acting on *resolve*. Both fit the same poller later.
- **Log every decision** to the app log with a `[monitor]` prefix: the first
  check of a reference (baseline), a catch-up window longer than a few minutes,
  each check that finds new incidents (reference, incident starts, armed tasks),
  every ok↔error transition with the error text, and what each queued fire
  ended as (agent started / delivered into a live session / delivery failed →
  notified / cancelled / dropped because the task closed). The status file only
  holds the latest state, so the log is the only overnight history.
