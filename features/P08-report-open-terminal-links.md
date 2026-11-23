---
id: P08
epic: P — Agent-facing control API
title: "Report → terminal links — click a task/subgoal in a web report to focus its terminal (or start it) and move its tab to first"
size: S
requires: [P07, P02, F07]
novel: true
---

## What
A local web report (P07) can mark any element as a **terminal link**:

```html
<a href="#" data-tm-task="I425" data-tm-goal="g2_5_3">g2_5_3</a>   <!-- a subgoal -->
<a href="#" data-tm-task="I727">I727</a>                            <!-- the whole task -->
```

Clicking it brings the TaskMan window to the front and does exactly what the app's own
"open terminal" button does for that task/subgoal: if its terminal is already open,
**focus** that tab; if not, **open it and start the agent** with the usual
task/subgoal brief (the same path as P02 `open-terminal`, including the account and
agent defaults). In the same action the tab moves to the **first position** of the
terminal strip (top-left) — the same reorder as the F07 tab menu's "Move to first", so
the new order persists like any other reorder and shows in every strip (F27). The
report's top bar (P07) shows the outcome ("opened I425 › g2_5_3"
or the error).

The report script only writes the `data-tm-*` attributes; the click handling lives in
the top bar the app injects, so every report gets it for free.

## Why it exists
A report like "my open PRs, with the task/subgoal each belongs to and whether that
terminal is running" is a to-do list: the next move after reading a row is to go to
that subgoal's session. Copying the id into the app and hunting for the subgoal turns
a one-click action into a context switch. And a terminal opened (or found) from a
report should land where the eye goes first — tacked on at the far right of a long strip,
or left wherever it was, it still has to be hunted for. The user clicking a link in a page the app
itself served is an explicit human action, so it doesn't need P01's Approve/Deny gate —
but it must not be triggerable by any other web page.

## Acceptance criteria (EARS)
- When the user clicks an element carrying `data-tm-task` (and optionally
  `data-tm-goal`) on a P07 report page, the page shall send a same-origin
  `POST /web/open` with `{taskId, subgoalId?}` and shall not navigate.
- When the app receives a valid `/web/open` for a task/subgoal whose terminal tab is
  open, the app shall focus that tab and bring its window to the front.
- When the app receives a valid `/web/open` for a task/subgoal with no open terminal,
  the app shall open one and start the agent with the normal task/subgoal brief (as P02
  `open-terminal` does after approval), and bring the window to the front.
- When the app handles a valid `/web/open`, the app shall move the focused or newly
  opened terminal tab to the first position of the shared tab order (F07 "Move to first"
  / F27), in the same action as the focus/open, for both the already-open and the
  newly-opened case.
- When a tab was moved to first by `/web/open`, the app shall persist the new order the
  same way as an F07 reorder, so it survives a restart and shows first in every strip
  that holds the tab (main, filtered, task panel).
- When the tab is already first, the app shall leave the order unchanged.
- The app shall move the tab only for `/web/open`; P02 `open-terminal` requests from
  the control API, autorun and the app's own open buttons shall keep their placement.
- The app shall not show the Approve/Deny banner for `/web/open`; it shall append the
  call to the control-API audit log with `source: "web-report"`.
- If the task id doesn't exist, or the subgoal id isn't in that task's v2 document,
  then the app shall respond with an error and open nothing; the top bar shall show it.
- The app shall accept `/web/open` only when: `Host` is loopback on its port, `Origin`
  is `http://127.0.0.1:<port>` or `http://localhost:<port>`, `Sec-Fetch-Site` is
  `same-origin` (when sent), the body is `application/json`, and it carries the
  per-boot web token the app embedded in the page's top bar. Anything else → 403.
- The top bar shall show "opened <task> › <subgoal>" on success and the error text on
  failure.

## Build notes
- The web token is minted per app boot, kept in memory, and written only into report
  pages the app serves (inside the injected top bar), never into `reports.json` or the
  report script's output.
- Requiring `application/json` makes a cross-origin `fetch` need a CORS preflight; the
  server sends no CORS headers, so a foreign page can't even send it — the Origin and
  token checks are a second and third layer.
- Reuse the P02 renderer path (`open-terminal` exec) without the approval step: build a
  request record, audit it, and dispatch it to the renderer directly.
- Move-to-first = the F07 reorder of the tab onto the **globally** first tab of the
  shared order (not the first tab of one filtered strip), so it is first in every strip.
  A new tab is appended first and then moved; do it right after the open, in the same
  renderer exec, so no frame shows it at the end. Mark the request (e.g. `toFront: true`)
  in the web handler only, so other P02 callers are unaffected.
- Bring the window forward with restore-if-minimized + `show()` + `focus()` (macOS:
  `app.focus({ steal: true })`).
