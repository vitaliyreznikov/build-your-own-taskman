---
id: P09
epic: P — Agent-facing control API
title: "Report actions — buttons in a web report that run the report's own declared action script, then refresh the page"
size: S
requires: [P07, P08]
novel: true
---

## What
A P07 report can declare **actions** — named operations its own script (or another
script in the data root) performs on the user's behalf — and render buttons for them:

```json
{ "id": "my-prs", "title": "My open PRs", "script": "I353/audit.py", "args": ["--html"],
  "actions": { "attach": { "script": "I353/audit.py", "args": ["--action", "attach"] } } }
```

```html
<button data-tm-action="attach"
        data-tm-params='{"task":"I736","url":"https://github.com/BoltApp/source/pull/76653"}'>
  Attach to I736</button>
```

Clicking the button makes the injected top bar `POST /web/action` with
`{report, action, params}`. The app looks the action up **in `reports.json` for that
report** (a page can only name actions its report declared), runs the action's script
with `params` as **JSON on stdin** (never as argv or through a shell), and on success
re-runs the report in place (the P07 Reload), so the result is visible immediately. The
top bar shows the script's first stdout line, or its error.

## Why it exists
A report that finds a problem ("this PR isn't attached to a task, but I753's note
mentions it") is most useful when fixing it is one click away. The fix is
data-specific — only the report's author knows that "attach" means "append a
`blocked-by-pr` row" — so the app doesn't learn the operation; it only provides a
safe, same-origin way for a page it served to invoke a script the user registered.

## Acceptance criteria (EARS)
- When the user clicks an element with `data-tm-action` on a report page, the page
  shall send `POST /web/action` with `{report: <report id>, action, params}` where
  `params` is the element's `data-tm-params` JSON (or `{}`), and shall disable the
  element until the call finishes.
- When the app receives a valid `/web/action`, it shall run the action's `script` (data
  root relative, refused outside it) with its `args`, cwd = data root, `params` as JSON
  on stdin, and a 30 s timeout.
- If the action isn't declared for that report in `reports.json`, then the app shall
  respond 404 and run nothing.
- When the action script exits 0, the page shall show its first stdout line in the top
  bar and re-run the report in place; when it exits non-zero, the page shall show the
  stderr tail and not refresh.
- The app shall apply P08's checks to `/web/action` (loopback Host, same Origin,
  `Sec-Fetch-Site: same-origin`, `application/json`, per-boot web token) and append
  each call to the control-API audit log with `source: "web-report"`.
- The action script shall validate `params` itself; the app passes them through
  unmodified and never interpolates them into a command line.

## Build notes
- Reuse P07's runner (execFile, no shell) with stdin input; keep the action registry
  lookup on every request (re-read `reports.json`).
- The page knows its report id from the top bar (`data-tm-report`), injected by the app.
- Actions should be idempotent where possible (e.g. attach = append only if the row
  isn't there), because a double click or a retry may run them twice.
