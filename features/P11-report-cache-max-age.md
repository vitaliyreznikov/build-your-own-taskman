---
id: P11
epic: P — Agent-facing control API
title: "Report cache max-age — a report may declare how long its last result stays fresh; opening it within that window serves the cache and runs nothing"
size: S
requires: [P07, P09]
novel: true
---

## What
A P07 report entry in `reports.json` may set an optional **`maxAgeSec`**:

```json
{ "id": "my-prs", "title": "My open PRs", "script": "I353/audit.py", "args": ["--html"],
  "maxAgeSec": 3600 }
```

When the report is opened (`GET /reports/<id>`, including F5) and its last successful
result (`.reports-cache/<id>.html`) is **younger than `maxAgeSec`**, the app serves that
result as-is and **does not run the script**. The injected top bar says how old it is
("updated 14:02 · 12 min ago · cached") and its **Reload** button still forces a fresh
run. Once the cache is older than `maxAgeSec`, missing, or the entry has no
`maxAgeSec`, the page behaves exactly as in P07: previous result shown instantly plus
a background fresh run.

## Why it exists
P07's "re-run on every open" is right for cheap, local data — and expensive for a
report that calls a rate-limited API. A PR-audit report that makes dozens of GitHub
calls per run, opened a few times an hour, burns through the account's shared hourly
quota (the same pool the app's PR-blocker polling and every other `gh` user draw
from), and then *everything* that talks to GitHub fails. The report's author knows
how stale its data may be; letting them say so per report keeps the cheap reports
live and the costly ones bounded, with the explicit Reload as the escape hatch.

## Acceptance criteria (EARS)
- The app shall accept an optional `maxAgeSec` on a `reports.json` entry: a positive
  number of seconds. An entry with a missing, non-numeric, zero or negative
  `maxAgeSec` shall behave as if it had none (P07 behavior); it shall not be rejected.
- When `GET /reports/<id>` (without `?fresh`) is requested for a report with
  `maxAgeSec`, and `.reports-cache/<id>.html` exists with a modification time less than
  `maxAgeSec` seconds ago, the app shall return that cached page with the top bar and
  shall not run the report's script nor schedule a background refresh.
- While serving a cached page under `maxAgeSec`, the top bar shall show when the
  result was produced and its age (e.g. "updated 14:02 · 12 min ago · cached").
- When the user clicks **Reload** on any report page, the app shall run the script
  (`?fresh=1`) regardless of `maxAgeSec`.
- When a P09 action succeeds, the page's refresh shall run the script regardless of
  `maxAgeSec` (the page after an action must never show the pre-action cache).
- If the cache is missing or at least `maxAgeSec` old, or the report has no
  `maxAgeSec`, then the app shall behave exactly as P07 (previous result + loading bar
  + background `?fresh=1` run).
- When a fresh run succeeds, the cache file's modification time shall become the
  start of the new freshness window (the cache is rewritten as in P07).
- The home page (`GET /`) shall show a report's `maxAgeSec` (e.g. "cached 60 min").

## Build notes
- Freshness is the cache file's mtime — P07 already writes it on every successful run
  and it survives restarts, so no new state is needed.
- The fresh path (`?fresh=1`) is untouched: Reload and the P09 post-action refresh
  already go through it, which is what makes both bypass the gate for free.
- Render the age when the page is served; it does not need to tick live.
