---
id: J01
epic: J — Claude account usage
title: Claude usage-limit widget (pace-relative)
size: M
requires: [B01]
novel: false
---

## What
A small always-visible widget in the left sidebar that shows the state of the
signed-in Claude account's two rolling usage limits — the **5-hour session** limit
and the **7-day (weekly)** limit. For each window it shows three things on one line:

```
5h:  34% +3%  · resets in 2h10m
1wk:  8% −4%  · resets in 4d 6h
```

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
- The system shall display, for both the 5-hour and the 7-day window, the percent
  of the limit consumed.
- The system shall display, for each window, a pace delta computed as the window's
  elapsed-time percent minus its consumed percent, signed so that a positive value
  means consumption is behind the time-pace (headroom) and a negative value means
  consumption is ahead of the time-pace (burning fast).
- The system shall visually distinguish the ahead-of-pace (negative) state from the
  behind-pace (positive) state.
- The system shall display, for each window, a countdown to when the window resets.
- When a window is fully consumed, the system shall indicate the limit is reached
  rather than showing a misleading delta.
- The system shall refresh the underlying usage figures on a periodic schedule and
  shall re-render the countdown and pace delta continuously between refreshes
  without issuing a network request for each tick.
- When usage data cannot be fetched (no credentials, network error, expired token),
  the system shall show an unobtrusive unavailable/error state rather than stale
  numbers presented as current.

## Build notes
- Source of truth is Claude Code's own usage endpoint —
  `GET https://api.anthropic.com/api/oauth/usage` with
  `Authorization: Bearer <accessToken>` and header
  `anthropic-beta: oauth-2025-04-20`. The response carries `five_hour` and
  `seven_day` objects, each with `utilization` (0–1) and `resets_at` (ISO
  timestamp). There is **no CLI subcommand** for this; the HTTP call is the only
  interface.
- The OAuth access token is whatever the local Claude Code install already holds —
  on macOS it lives in the login Keychain under service `Claude Code-credentials`
  (JSON → `.claudeAiOauth.accessToken`). The widget reads it fresh each poll rather
  than caching it, since Claude Code rotates it. Token access is a
  main-process/back-end concern, never the renderer.
- Split the two update rates: poll the network infrequently (the utilization only
  moves when the account actually makes requests — ~60s is ample), but recompute the
  countdown and pace delta on a fast local tick (~1s) from the cached `resets_at`
  plus the fixed window length (5h / 7d). Elapsed-time percent =
  `(now − (resets_at − windowLength)) / windowLength`, clamped to 0–100.
- The pace delta is the novel bit: it is a *derived* comparison of two percentages
  (time vs. quota), not anything the API returns. Keep the sign convention in one
  place so the UI and any thresholds agree.
- Degrade quietly: an expired token or offline machine should read as "unavailable",
  never as 0% used.
