# PROFILES — ready-made bundles

Don't want to hand-pick? Start from one of these. Each is already
prerequisite-closed — paste it straight into [`prompts/01-build.md`](prompts/01-build.md).

## 1. Minimal Markdown Kanban
A plain, agent-legible task board. No terminals, no status system. The smallest
thing that captures the "tasks are Markdown files" idea.

**Features:** A01–A07, A09, B01–B06, B10, B11
**You get:** Markdown-file tasks, a Kanban board, drag-between-columns, a note
editor, NextAction sorting, git-synced storage. An LLM can read and edit every
task by pointing at its file.

## 2. Solo-agent desk
One agent, working where the task lives. Adds the in-app terminal and reusable
prompts on top of the minimal board.

**Features:** Profile 1 **+** B12, B13, F01–F03, F09, G01, G02
**You get:** open a terminal on a task, it auto-moves to IN PROGRESS, insert
reusable prompt parts with ⌘K. This is the single-operator workflow.

## 3. Parallel-agent cockpit  ·  the full thesis
Run many agents at once and supervise by attention. This profile *is* the point
of the whole project — "one human reviewing many parallel agents."

**Features:** everything — A*, B*, C*, D*, E*, F*, G*, H*, I*
**You get:** multi-terminal tasks with tabs, live per-agent status icons
(🤖/⏳/⚠️/✅), desktop notifications only when an agent needs *you*, one-keystroke
jump to the next agent that's stuck, session history + resume, boards for
organizing parallel work, and a blocks/spawned relation graph.

---

### Sizing note
Profile 1 is an afternoon. Profile 2 adds the terminal (the one genuinely hard
piece — tmux + xterm + pty lifecycle). Profile 3 adds the Claude-hook status
layer, which depends entirely on Profile 2's terminal existing. Build in that
order even if you want the whole thing.
