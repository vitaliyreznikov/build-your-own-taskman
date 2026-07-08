# Prompt 01 — Build (the runbook / "constitution")

Paste this once at the start of a build session, with your resolved build order
from `prompts/00-resolve.md` and the relevant `features/*.md` files attached.

---

You are building a personal task manager from a spec. This is the constitution —
follow it for every feature.

**Principles (non-negotiable):**
1. **The Markdown files are the source of truth.** State lives in flat
   Markdown/JSON files, not a database. Anything an LLM might need to read or edit
   must be a plain file. This is the whole point of the app.
2. **Build in the given order.** The plan is topologically sorted; don't skip
   ahead of a prerequisite.
3. **One feature at a time.** Implement it, satisfy its acceptance criteria, make
   the app runnable, then move on. Stop at each milestone and let me verify.
4. **Match the acceptance criteria exactly** — they're written in EARS form
   ("when X, the system shall Y"). Treat each as a test.
5. **Don't over-build.** Prefer the simplest thing that satisfies the criteria.
   No feature not on the resolved list.

**Stack (change freely if you prefer — the spec is stack-agnostic):** the
reference app is Electron + Vite + React + TypeScript, with tmux + xterm.js for
terminals. Pick whatever you and I agree on; the specs describe behavior, not a
framework.

**For each feature file:** read `## What`, `## Why it exists`, `## Acceptance
criteria`, `## Build notes`. The build notes carry real edge cases the original
implementation hit — honor them.

**Working style (recommended, this is the app's own philosophy):** work in an
isolated branch/worktree, open a PR, and don't merge or restart anything without
my say-so. If you can, report status as you go with markers:
`🤖 working:` / `⏳ waiting:` / `⚠️ attention:` / `✅ done:`.

Start with the first feature in the build order. Confirm the plan back to me
before writing code.
