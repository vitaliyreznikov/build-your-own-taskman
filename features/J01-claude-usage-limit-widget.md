---
id: J01
epic: J — Claude account usage
title: Claude usage-limit widget (pace-relative)
size: M
requires: [B01]
novel: false
---

## What
A small widget at the top of the left sidebar (above the View section) that shows
the state of the signed-in Claude account's rolling usage limits — the **5-hour
session** limit, the **7-day (weekly)** limit, and the **per-model weekly cap**
(e.g. Fable). Each window is shown on two stacked lines (so the narrow sidebar
never scrolls horizontally): a first line with used %, pace delta, and a second
line with the reset countdown:

```
5h    9% +6%
resets in 4h14m
1wk   2% +29%
resets in 4d 20h
Fable 0% idle
```

A per-model weekly window whose reset time is null (the model hasn't been used
this week) shows just its percent and an `idle` marker — no delta or countdown,
since there is no active window clock to measure against.

- **used %** — how much of the window's quota is consumed (from the API's
  `utilization`, 0–1, rendered as a percent).
- **pace delta** — `time_elapsed% − used%`, i.e. how the consumption compares to
  where an even, linear burn would have you at this point in the window.
  - **`+` (positive)** means you are *behind* the clock — using slower than time is
    passing, so you have headroom. Shown as the reassuring state.
  - **`−` (negative)** means you are *ahead* of the clock — burning faster than the
    window elapses, at risk of hitting the cap before it resets. Shown as the
    warning state.
- **reset countdown** — relative time until the window resets (`resets_at`).

## Why it exists
Claude subscriptions cap usage on a 5-hour session window and a 7-day window, but
those numbers are invisible unless you open a session and run a slash command. When
you drive long agent sessions from TaskMan you want to know, at a glance and without
interrupting anything, whether you're about to run out — and crucially *whether your
current burn rate is sustainable for the rest of the window*. A raw "34% used" says
little on its own; "34% used but only 31% of the window has elapsed" tells you
you're running slightly hot. The pace delta turns a static gauge into a
rate-of-consumption signal so you can throttle before you hit a wall.

## Acceptance criteria (EARS)
- The system shall display, for the 5-hour, the 7-day, and the per-model weekly
  window, the percent of the limit consumed.
- Where a window has no active reset time (e.g. a per-model cap the account hasn't
  touched this week), the system shall show its percent only, without a pace delta
  or countdown.
- The system shall display, for each window, a pace delta computed as the window's
  elapsed-time percent minus its consumed percent, signed so that a positive value
  means consumption is behind the time-pace (headroom) and a negative value means
  consumption is ahead of the time-pace (burning fast).
- The system shall visually distinguish the ahead-of-pace (negative) state from the
  behind-pace (positive) state.
- The system shall display, for each window, a countdown to when the window resets.
- When a window is fully consumed, the system shall indicate the limit is reached
  rather than showing a misleading delta.
- The system shall re-render the countdown and pace delta continuously without
  issuing a network request for each tick.
- The system shall refresh the underlying usage figures in response to user
  activity once the figures exceed a staleness age, and shall not issue network
  requests while the app is idle/unattended.
- When the figures are stale (no recent activity has refreshed them), the system
  shall visually mark them as stale and offer a manual refresh control.
- When usage data cannot be fetched (no credentials, network error, expired token),
  the system shall show an unobtrusive unavailable/error state — never stale numbers
  presented as current — and shall offer a manual refresh control.

## Build notes
- Source of truth is Claude Code's own usage endpoint —
  `GET https://api.anthropic.com/api/oauth/usage` with
  `Authorization: Bearer <accessToken>` and header
  `anthropic-beta: oauth-2025-04-20`. The response carries `five_hour` and
  `seven_day` objects, each with `utilization` and `resets_at` (ISO timestamp).
  **`utilization` is a percentage on a 0–100 scale** (e.g. `6.0` == 6% used, and
  it matches the corresponding `limits[].percent`), *not* a 0–1 fraction — it can
  exceed 100 for an over-limit account with extra usage. Normalize it once at the
  boundary. There is **no CLI subcommand** for this; the HTTP call is the only
  interface.
- The per-model weekly cap comes from the response's `limits[]` array — the entry
  with `kind: "weekly_scoped"` whose `scope.model.display_name` matches the model
  (e.g. "Fable"), carrying `percent` and a possibly-null `resets_at`.
- The OAuth access token is whatever the local Claude Code install already holds —
  on macOS it lives in the login Keychain under service `Claude Code-credentials`
  (JSON → `.claudeAiOauth.accessToken`). The widget reads it fresh each poll rather
  than caching it, since Claude Code rotates it. Token access is a
  main-process/back-end concern, never the renderer.
- Split the two update rates, and make the network side activity-gated rather than
  a blind timer: the utilization only moves when the account actually makes
  requests, and an unattended app should make no requests at all. Track user
  activity (pointer/keyboard/focus/visibility) and trigger a refresh only once the
  cached figures pass a staleness age (~2 min). Independently, recompute the
  countdown and pace delta on a fast local tick (~1s) from the cached `resets_at`
  plus the fixed window length (5h / 7d) — no network per tick. Elapsed-time
  percent = `(now − (resets_at − windowLength)) / windowLength`, clamped to 0–100.
- Staleness is simply "older than the refresh age", which — because activity
  refreshes it — also means "no recent activity". Surface it as a dimmed state plus
  a manual refresh button so a returning user can force an immediate update.
- The pace delta is the novel bit: it is a *derived* comparison of two percentages
  (time vs. quota), not anything the API returns. Keep the sign convention in one
  place so the UI and any thresholds agree.
- Degrade quietly: an expired token or offline machine should read as "unavailable",
  never as 0% used.
