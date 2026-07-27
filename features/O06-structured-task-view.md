---
id: O06
epic: O — Structured task documents (v2)
title: Read-only structured task view
size: M
requires: [O01, O03, B06]
novel: false
---

## What
For a task whose type is `v2` (O02), the detail panel renders the document as
**read-only structured sections** instead of a Markdown editor:

1. **Subgoals** — the tree (O03), collapsible per node, each node showing a state
   marker (`todo` / `doing` / `done`), one **chip per blocker**, and for `pr`
   blockers the **live review state** from the poller (O07 / L01).
2. **body_human** — "what should be done", rendered Markdown.
3. **body_state** — "where this stands now", rendered Markdown.
4. **KB** (O04) — a list of **collapsed questions**; expanding one reveals the
   answer.
5. **Timeline** (O05) — a feed, **newest-first**.

The human is a **reader** of a v2 task. There is no in-app structured editing: no
subgoal add/edit/reorder, no state toggles, no blocker forms, no body editor. Two
escape hatches guarantee nothing is ever invisible or unreachable:

- **Open in editor** — hand the raw `I<N>.json` (or a side file) to the external
  editor, exactly as B07 does for a note.
- **Raw document fallback** — show the underlying JSON/JSONL text on demand, and
  when a document **fails to parse**, show an explicit error state (with the parse
  reason and the raw text) rather than an empty task.

The side files load **lazily** — the KB and timeline are read only when their
section is opened. The v1 Markdown view/edit path (B06) is untouched.

## Why it exists
The v2 document exists because agents write the task and the human reviews it. The
structured view is that review surface: it answers "what is left, what is stuck, and
what changed" without reading a scroll, which is precisely what the flat note could
not do.

**Read-only is a design decision, not an unfinished feature.** Building structured
editors — tree drag-and-drop, blocker dialogs, state pickers — would be the bulk of
the work in this epic and would compete with the thing that actually writes these
documents: the agent. Two writers with different mental models produce merge
conflicts against a file on disk and a UI that has to guess whether an in-flight
edit or the agent's write wins. Meanwhile the human already has a better editor for
the rare hand-edit: their editor. So the app optimizes for reading, the agent
optimizes for writing, and the escape hatch covers the remainder.

The fallbacks matter more than usual here because untrusted writers touch the data
(O01). A structured view that renders nothing when the JSON is malformed would hide
the task *and* hide the reason — so a parse failure is a visible, explained state
with the raw text one click away, and the fix is always possible from inside the app.

Lazy loading keeps opening a task cheap: a mature task's KB and timeline are the two
files that grow without bound, and most opens only want the subgoals.

## Acceptance criteria (EARS)
- When the user opens a task whose type is `v2`, the system shall render the
  structured view and shall not render the Markdown note editor.
- When the system renders the structured view, it shall show the subgoal tree with a
  state marker per node and one chip per blocker on that node.
- When a subgoal has a `pr` blocker, the system shall show that PR's live review
  state on its chip.
- When a subgoal has children, the system shall let the user collapse and expand that
  node.
- When the user opens the KB or timeline section, the system shall load that side file
  at that point and shall not read it while the section is collapsed.
- When the system renders the KB, it shall show questions collapsed and shall reveal
  the answer when a question is expanded.
- When the system renders the timeline, it shall order entries newest-first.
- While a v2 task is open, the system shall offer "open in editor" for the raw
  document and shall offer a raw-document view.
- If the document fails to parse, then the system shall show an error state naming
  the failure and offering the raw text and the editor, and shall not show an empty
  task.
- The system shall not provide in-app creation, editing, reordering, or state changes
  of subgoals, blockers, KB entries, timeline entries, or bodies for a v2 task.
- When a v1 task is opened, the system shall render and edit its Markdown note
  exactly as before.

## Build notes
- New component `StructuredTaskView`, selected by `detailKind(taskId)` (O02) at the
  same branch point where `NotePanel` (B06) is chosen. `NotePanel` gains no v2 code.
- Render `body_human` / `body_state` with the same Markdown renderer and the same
  external-link allowlist (B08) the note panel uses — they are Markdown strings, and
  they contain links to PRs, dashboards and files.
- The file watcher (B11) must treat `I<N>.json` and the two side files as task data:
  a v2 task open on screen should show the "changed on disk" affordance when an agent
  writes it mid-session. This is the normal case here, not the exception.
- Never do offset-based markdown surgery on a v2 document. The v1 panel can toggle a
  `- [ ]` checkbox by byte offset; there is no equivalent for a v2 subgoal, and the
  read-only rule means you don't need one.
- Lazy load with a per-section cache keyed by mtime, and drop it on a watcher event.
  Tail the timeline (last N entries) with a "load older" control instead of reading a
  large file into the DOM.
- Blocker chips: distinct affordance per kind — `pr` (click opens the PR, shows
  review state), `answer` (who + how long waiting), `user-action` (who + what).
  Unknown kinds get a neutral chip (O03).
- Show elapsed time for `answer` / `user-action` blockers; a stale `answer` blocker is
  the thing the human is supposed to notice from this screen.
- Collapse state is view state — keep it in the renderer, never write it back into
  the document. Writing UI state into a git-synced file makes every scroll a commit.
- The raw view is also the debugging tool for the forgiving parser: showing which
  records were dropped, and why, next to the raw text turns a silent drop into a
  visible one.
