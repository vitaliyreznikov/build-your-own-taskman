---
id: P07
epic: P — Agent-facing control API
title: "Local web reports — script-backed HTML pages served by the app, recomputed on every browser refresh"
size: M
requires: [P01]
novel: true
---

## What
The app's loopback HTTP server (P01) also serves a small **read-only website** for
the user's browser:

- **`GET /`** — the TaskMan home page: a list of the registered reports (title,
  owning task, script, how long its last run took and when).
- **`GET /reports/<id>`** — runs that report's script **right now** and returns its
  stdout as the page. Pressing F5 in the browser re-runs the script, so the page is
  always computed from live data — there is no cache and no "regenerate" button.

A **report** is a script an agent or the user wrote inside the data root (usually in
a task's detail directory) that prints a complete HTML page to stdout. Reports are
registered in a git-synced data file, `reports.json`, at the data root:

```json
[
  { "id": "my-prs", "title": "My open PRs", "task": "I353",
    "script": "I353/audit.py", "args": ["--html"] }
]
```

`id` is the URL slug (`[a-z0-9-]`), `task` is optional (links the report to the task
that owns it), `script` is a path relative to the data root, `args` optional. The
script is executed directly (it must be executable, shebang decides the interpreter)
with the data root as its working directory.

## Why it exists
Agents increasingly produce *status views* — "every open PR, grouped by review
state, with the task and whether its terminal is live" — and the honest home for
such a view is a script, because the data is live (GitHub, tmux, the board). Writing
the result into a note or a static HTML file makes it stale the moment it's written,
and a hosted artifact adds a publish step and an external copy. A local page that
re-runs the script on refresh is the smallest thing that is always current: the
agent's job is to keep the script correct; the user's job is to press F5.

It reuses the server P01 already runs, so there is no second port, process or
install.

## Acceptance criteria (EARS)
- When the browser requests `GET /` on the control API port, the app shall return an
  HTML page listing every valid entry of `reports.json` with a link to
  `/reports/<id>`, its title, owning task and last-run time/duration (if any).
- When the browser requests `GET /reports/<id>` for a registered report, the app shall
  execute the report's script with the data root as cwd and return its stdout as
  `text/html; charset=utf-8` with `Cache-Control: no-store`.
- When the same report is requested again (F5), the app shall run the script again;
  a previous result may be shown only while the new run is in progress (below), never
  instead of it.
- When `GET /reports/<id>` is requested and a previous successful result exists, the
  app shall respond immediately with that previous page plus a loading bar pinned to
  the top, then fetch `GET /reports/<id>?fresh=1` (which runs the script) and replace
  the page with the result once it arrives. With no previous result it shall show only
  the loading bar until the run finishes.
- The app shall keep the last successful output per report on disk (machine-local,
  gitignored `.reports-cache/<id>.html`) so the previous state survives an app restart.
- Every report page shall carry a **Reload** control in that top bar; clicking it shall
  show the loading bar and re-run the report in place without leaving the page
  (keeping the current content visible until the new one arrives).
- If a background refresh fails, then the page shall keep the previous content and
  show the failure in the top bar: the exit status plus the last line of the script's
  stderr, with the stderr tail (last few lines) on hover — so a failure is diagnosable
  without re-running the script by hand.
- When a report run fails, the app shall write one line to its log naming the report,
  the exit status and the stderr tail, and `GET /` shall show that last error under the
  report.
- While a run of a report is in flight, a concurrent request for the same report
  shall wait for and share that run's result rather than start a second one.
- If the script exits non-zero, times out (60 s) or the entry is invalid, then the app
  shall return a 500 HTML page showing the exit status and the script's stderr tail.
- If `reports.json` is missing, unreadable or invalid JSON, then `GET /` shall still
  render, showing the problem instead of the list.
- If an entry's `script` resolves outside the data root, then the app shall refuse to
  run it.
- The `GET /` and `GET /reports/*` routes shall not require the API token (a browser
  cannot send it); every other route keeps P01's token guard.
- If a request's `Host` header is not `127.0.0.1:<port>` or `localhost:<port>`, then
  the app shall reject it with 403 (DNS-rebinding guard).
- If a request carries `Sec-Fetch-Site: cross-site` or `same-site`, then the app shall
  reject it with 403 without running anything (a web page cannot trigger a report).
- The app shall re-read `reports.json` on every request, so a newly registered report
  appears without an app restart.

## Build notes
- The top bar (loading line, "updated HH:MM", Reload) is injected by the app into
  whatever the script printed, so every report gets it for free; the script stays a
  plain "print an HTML page" program. Swap the content with
  `document.open(); document.write(html); document.close()` so the report's own
  inline scripts (e.g. a copy-to-clipboard button) run again.
- Implement as a handler module the P01 server calls **before** its token guard and
  that returns "not mine" for any other path — the API routes don't change.
- Spawn with `execFile`/`spawn` (no shell), `cwd = DATA_ROOT`, a timeout and a
  bounded stdout buffer (e.g. 20 MB). Keep the last run's `{at, ms, ok}` in memory for
  the home page; nothing is persisted.
- The home page is plain server-rendered HTML with light/dark tokens; no client JS is
  needed. HTML-escape every value taken from `reports.json`.
- A report script should also be runnable by hand (`<script> --html > out.html`), so
  the page and the cron/agent path share one code path.
- Optional affordance: the note of the owning task links
  `http://127.0.0.1:<port>/reports/<id>`; http links already open externally.
