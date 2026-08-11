---
id: A12
epic: A — Markdown data model
title: Reuse a recently-freed id (bounded window, terminal-guarded)
size: S
requires: [A07, F02, F14]
novel: true
---

## What
When a task is created, the allocator no longer always hands out `max + 1`. It
first looks at the **30 integers immediately below** the next monotonic id and, if
one of them is a **genuine gap** *and* **has no terminal bound to it**, reuses that
number instead. Only when no number in the window qualifies does it fall back to
`max + 1`.

Concretely, with `next = max + 1` (the monotonic id A07 would have handed out) and
a window `W = 30`, a candidate integer `N ∈ [next − W, next − 1]` is **reusable**
iff both hold:

1. **It's a gap** — no `tasks.md` row and no `I<N>.md` / `I<N>.json` document on
   disk claim `N`. (Same two sources A07 already unions, but tested per-number, not
   just for the max.) A gap means the task that once held `N` was deleted.
2. **No terminal is bound to it** — neither a live tmux session named
   `task-I<N>[…]` (F02's session→task mapping) *nor* an entry for `I<N>` in the
   durable `.terminals.json` snapshot (F14). The snapshot matters because a
   terminal can exist for an id while tmux is momentarily down (e.g. just after a
   reboot, before restore): handing that id to a new task would collide with the
   terminal the moment it comes back.

If several candidates qualify, reuse the **lowest** one (fill the oldest gap first,
keep the numbering dense, deterministic). If none qualify, allocate `next` exactly
as A07 does today — the monotonic path is the fallback, never removed.

This deliberately relaxes A07's "monotonically increasing and never reused" rule,
**but only inside the bounded, recency-limited window and only for numbers nothing
live still points at.** Ids older than the window are never recycled, so a
reference to an old `I<N>` keeps its meaning; the reuse only ever touches a number
freed in the last handful of tasks.

## Why it exists
Task numbers march upward forever under A07, so a board that churns through
short-lived tasks (a placeholder that F13 files then deletes when the work turns
out to belong to an existing task; a task created and immediately dropped)
accumulates ever-larger ids with permanent holes just below the top. The user
wants those just-freed numbers **reused** rather than burning a fresh id every
time — keeping the id space compact and the numbers small enough to say out loud.

A07's caution is still respected where it bites: it warns that reused ids break
stale references (relations, session history, **terminal session names**). The
terminal-session-name hazard is the sharp one — a live or restorable `task-I<N>`
terminal *is* a hard collision — so that is exactly the condition the reuse guard
checks. Bounding reuse to the last 30 ids keeps any residual stale reference (an
old relation row or session-history entry naming a since-deleted `I<N>`) both rare
and recent, which is the accepted trade for a compact id space.

## Acceptance criteria (EARS)
- When a task is created, the system shall, before allocating `max + 1`, search the
  window of the 30 integers immediately below `max + 1` for a reusable id.
- The system shall treat an integer `N` in that window as reusable only when no
  `tasks.md` row and no `I<N>.md`/`I<N>.json` document claim it, **and** no terminal
  is bound to it.
- When testing whether a terminal is bound to `N`, the system shall count **both** a
  live tmux session mapping to `I<N>` (F02) **and** an `I<N>` entry in the durable
  terminal snapshot (F14); an id whose terminal is merely detached or awaiting
  restore shall be treated as bound.
- When one or more window integers are reusable, the system shall allocate the
  lowest such integer.
- When no window integer is reusable, the system shall allocate `max + 1`, matching
  A07's monotonic behaviour exactly.
- The reuse window shall be bounded (default 30) and recency-limited: an id below
  the window shall never be reused regardless of whether it is a gap.
- Id allocation shall remain race-safe: the "claimed" set shall be derived from
  `tasks.md` and the on-disk documents read at allocation time (A07), never from an
  in-memory counter, so a note an external agent already wrote is still never
  clobbered.

## Build notes
- The change lives in `ids.ts`'s `nextId`, but the terminal signal comes from
  outside the pure store layer. Keep the gap arithmetic pure in `ids.ts` and
  **inject** the terminal-bound ids: extend `nextId(tasks, opts?)` to accept
  `{ reserved?: Set<number>; window?: number }`, where `reserved` is the set of
  integers that currently have a terminal. `ids.ts` computes the claimed-number set
  (from `tasks` ids + a directory scan for `I<n>.md|json`), scans `[next-window,
  next-1]` low→high, and returns the first `N` that is neither claimed nor reserved,
  else `I${next}`. This keeps `ids.ts` free of any `ptyManager`/electron dependency
  and unit-testable.
- Build `reserved` at the single mint site, `createTaskInternal` (main.ts): union
  `ptyManager.listSessions()` (live, via `taskIdOf` → `I<n>` → integer) with
  `readTerminalSnapshot()` (durable, F14). Parse each `taskId` back to its integer
  and add to the set. Pass it into `nextId`.
- `createTaskInternal` is the *only* caller of `nextId` (all four entry points —
  the `tasks:create` IPC, the P03 `create-task` control-API verb, and M01
  create-from-chat — funnel through it), so the guard is added in exactly one place
  and every create path inherits it.
- Default `window = 30`; make it a named constant so it is trivially tunable. A
  `window` of `0` (or an empty search result) degrades cleanly to A07's `max + 1`.
- Do **not** widen the terminal test to relations/session-history references — the
  design accepts those as the bounded, recent residual risk. Reusing a number is a
  product choice here; the terminal guard is the one hard-collision it must avoid.
