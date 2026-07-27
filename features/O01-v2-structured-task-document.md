---
id: O01
epic: O — Structured task documents (v2)
title: v2 structured task document (I<N>.json)
size: L
requires: [A01, A02, A06]
novel: true
---

## What
A task's detail can be a **structured JSON document** (`I<N>.json`) instead of a
free-form Markdown note (`I<N>.md`). The document is small and fixed in shape:

```json
{
  "v": 2,
  "id": "I246",
  "title": "Migrate the fleet to k8s 1.34",
  "subgoals": [],
  "body_human": "## What should be done\n…",
  "body_state": "## Where this stands\n…"
}
```

- `v` is the document version (`2`), `id` and `title` mirror the index row.
- `subgoals[]` is a tree of work items with blockers (O03).
- `body_human` is **what should be done** — the intent, the definition of done,
  the constraints. It is written once and revised rarely.
- `body_state` is **where this stands now, for the human** — the current picture, a
  briefing you can read cold. It is rewritten as the work moves.

Both bodies are Markdown strings, so prose stays prose. The two append-heavy parts
of a task do **not** live in this file: the task KB (O04) and the timeline (O05)
are separate line-oriented side files under the task's detail directory
(`I<N>/kb.jsonl`, `I<N>/timeline.jsonl`).

**The files are the API.** Agents read and write `I<N>.json` and its side files
with ordinary file tools — the same way they already hand-edit the index table.
There is no CLI, no MCP server, no agent-facing IPC, and no schema-validation gate
in front of the document.

## Why it exists
A free-form note is a great prompt and a poor status report. After a few weeks of
agent work a note becomes a scroll: intent, dated progress bullets, reusable
findings and open questions all interleaved, and the human has to re-read the whole
thing to answer "what is left and what is stuck?". The information was always
there in four distinct kinds — intent, current state, reusable facts, history — so
the document names them instead of blending them.

**Files-as-API is the deliberate part.** The obvious alternative is to expose a
tool surface — a `taskman` CLI or an MCP server the agent calls to mutate the
document. That would buy validation and lose the property the whole app is built
on: the data is *ordinarily editable*. Agents already edit `tasks.md` by hand and
are extremely good at reading and writing small JSON files; a tool layer adds a
second thing to install, version, authenticate and keep in sync with the schema,
and it breaks the moment the agent runs somewhere the app isn't. Keeping the file
canonical means the document works in a plain editor, in `git diff`, over SSH, and
with any agent — not just the one we shipped a client for.

The cost of that choice is that untrusted, unvalidated writers touch the data, so
the reader must be built for it: **forgiving parse, never throw**. A malformed
subgoal must cost you that subgoal, not the task.

> Design intent, in the author's words: *"the files are the API — the agent already
> edits the table by hand; it can edit a small JSON doc by hand too."*

## Acceptance criteria (EARS)
- When a v2 task is created, the system shall write `I<N>.json` containing at least
  `v: 2`, `id`, `title`, `subgoals`, `body_human`, and `body_state`.
- When the system reads a task document, it shall parse it inside a guarded parse
  and, on any parse error, shall report an error state for that task without
  throwing and without affecting other tasks.
- When a record inside the document fails its type guard (a subgoal, a blocker, a
  KB entry, a timeline entry), the system shall drop that record and render the
  rest of the document.
- When the system writes a task document, it shall preserve any unknown top-level
  keys it did not itself produce.
- When the system writes a task document, it shall write it atomically (A06) with
  stable key order and a trailing newline.
- When the document nests deeper than a configured maximum depth, the system shall
  stop descending at that depth rather than recursing without bound.
- When an external writer (agent, editor) changes `I<N>.json` on disk, the system
  shall read the new content on next read rather than serving a cached copy.
- The system shall not require any CLI, IPC, or server call for an agent to read or
  write a task document.

## Build notes
- One module owns the shape: types, type guards, `readDoc`, `writeDoc`. Nothing
  else in the app touches the JSON directly.
- `readDoc` contract: `JSON.parse` in `try/catch` → check `v === 2` → per-field
  guards with defaults (`subgoals: []`, bodies `''`) → return
  `{ ok: true, doc }` or `{ ok: false, reason, raw }`. It has no throwing path;
  callers never need a `try`. Log the reason once per mtime, not per read, or a
  broken document floods the log.
- `writeDoc` contract: read-modify-write. Start from the parsed document *including
  the keys you don't understand*, apply the change, serialize with a fixed key order
  (`v, id, title, subgoals, body_human, body_state`, then unknown keys), two-space
  indent, trailing newline, then route through the atomic writer (A06). Stable
  serialization is what keeps `git diff` a one-line diff instead of a reshuffle.
- Bodies are Markdown *inside JSON strings*. Render them with the same Markdown
  renderer the v1 panel uses. Never do offset-based surgery on a v2 document — the
  v1 trick of toggling a `- [ ]` checkbox by byte offset in the note has no
  analogue here; a v2 state change is a parse → mutate → serialize round trip.
- Keep the document small. The two parts that grow without bound (facts learned,
  events) are side files precisely so the document you rewrite stays a few KB and
  an append never rewrites the whole thing.
- Depth cap ~10 on subgoal recursion, applied in the guard, so a cyclic or
  pathological document cannot hang a render.
- Side files are optional. A v2 task with no `I<N>/kb.jsonl` is normal; absence is
  an empty list, not an error.
