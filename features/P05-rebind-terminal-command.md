---
id: P05
epic: P — Agent-facing control API
title: "`rebind-terminal` command — a worker re-labels its tab onto a different task (I402 → I400)"
size: M
requires: [P01, P02, H08, F09]
novel: true
---

## What
A control-API verb that lets a running **task worker** move its own terminal from
one task to another: the tab that read `I402` now reads `I400`, and the terminal's
per-terminal status icon, its IN PROGRESS card, and its restore snapshot all follow
to `I400`. The underlying tmux session, its history, and the task's own files
(`I402.md`, `I402.json`, …) are left untouched — this is a **re-bind of the live
terminal**, not a rename of the task.

CLI shape:

```
taskman rebind <taskId>
```

- `action = "rebind-terminal"`, `params = { taskId, termId? }`.
- `termId` defaults to the caller's own terminal — the `TASKS_TERM_ID` the CLI
  already exports as `source.termId` (P01) — so a worker just says the task it now
  belongs to. An explicit `termId` re-binds a named terminal instead.
- `taskId` is the task the terminal should now belong to; it **must already exist**
  (same existence check as `open-terminal`, P02). Re-binding never invents a task.

Approval sentence (P01's `describe`): *"Session in `<source.taskId>` wants to
re-label its terminal onto `<taskId>`"* — with *"(no caller)"* when the source task
is absent, and naming the explicit `termId` when one is passed instead of the
caller's own.

On approve, the executor records a **re-bind overlay** for that session name and
adds the target task to **IN PROGRESS** (F09), exactly as opening a terminal does.
The request resolves `done` with `result = { termId, taskId }`.

## Why it exists
`open-terminal` (P02) reserves a task id **up front** so a new-task tab reads
`I<N>` from the first frame and never has to be renamed once the real task lands
(F13). But a worker can only learn *mid-session* that the work in front of it
actually belongs to a different, already-filed task — the new-task interview
concludes "this is really I400", or an agent picks up the wrong card. Until now the
tab was stuck showing the id baked into its tmux session name, because **every**
`sessionName → taskId` derivation (the tab label, the card grouping, the status
key, the live-terminal test) reads that name.

The status key is the hard constraint that shapes the design. The per-terminal
status hook (H08) bakes its key from `process.env.TASKS_TERM_ID` at the moment
`claude` launches; nothing outside the process can change that env afterwards, so a
true tmux `rename-session` would silently orphan a live worker's status. `rebind`
therefore keeps the stable session name and layers a re-association on top that
every id-resolver consults — which moves the label, card, status and snapshot in
one coherent step without touching the running process.

## Acceptance criteria (EARS)
- The system shall register a `rebind-terminal` action taking `taskId` and optional
  `termId`, and shall reject a request whose `taskId` names no task.
- When `termId` is omitted, the system shall re-bind the caller's own terminal
  (`source.termId`); when the resolved terminal names no live session, the request
  shall be rejected.
- When `rebind-terminal` is approved, the system shall record a durable re-bind from
  that session name to `taskId`, and shall move `taskId` into IN PROGRESS (F09).
- After a re-bind, the tab label, the per-terminal status icon, the terminal's card
  membership, and the reboot-restore listing shall all show the terminal under
  `taskId`, while the tmux session name, its transcript, and every task file remain
  unchanged.
- The re-bind shall survive an app reload and a reboot-restore of the terminal set,
  because it is keyed on the stable session name that both outlive.
- Re-binding a terminal back to the task its session name already encodes shall
  clear the overlay (a no-op re-bind is how you undo one).
- The approval description shall name the requesting session, the target task, and
  an explicit `termId` when one is passed.
- When the request resolves `done`, its result shall carry the terminal id and the
  new task id.

## Build notes
- **Overlay, not rename.** Persist a machine-local, gitignored map
  `{ [sessionName]: taskId }` (a sibling of the F14 snapshot and the H08 status
  file — none of these are synced). Load it in main, push it to the renderer on its
  own channel the way live status is pushed, and re-read it on reload.
- **One resolver.** The overlay is honoured wherever a session name (or a status
  key derived from one) becomes a task id: the tab label, `taskStatuses` /
  `taskTerminals` grouping, `statusForTab`, and — in main — `taskHasLiveTerminal`
  and the A12 `reservedTerminalIds` set. Keep the un-rebound `statusKeyRef` pure;
  apply the overlay in a thin wrapper at those call sites so a session with no
  overlay behaves exactly as before.
- **Status keeps flowing under the old key.** The running hook still writes under
  the session name (`task-I402`); the tab's exact-name lookup still hits, and the
  card sees it because the grouping remaps `task-I402`'s derived id to `I400`. No
  status is lost and none has to be migrated.
- **IN PROGRESS** reuses P02/F09's "add on open" move for the target; the source
  task is left on the board (the board's memberships are removed by hand, by
  contract), so a stray placeholder id is the user's to clear — the verb changes
  *nothing else*.
- **Index/subgoal.** v1 re-binds at task level and keeps the index/subgoal slug
  parsed from the original name; if the target task already owns a terminal at the
  same index the tab still reads correctly (exact-name status lookup), and the rare
  index tie is left as a known limitation rather than renaming the session.
- **Undo** is a re-bind to the session's own native id, which removes the map entry
  rather than writing an identity mapping.
