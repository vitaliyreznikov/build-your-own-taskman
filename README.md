# build-your-own-taskman

**A task manager, shipped as a spec — not as code.**

This repo is not source you clone. It's a **menu of features with a dependency
graph**. You tick the features you want, run one prompt that resolves the
prerequisites, and hand the result to an LLM (Claude Code, Cursor, Copilot,
Gemini CLI…). The agent builds *your* version of the app.

The spec is the product. Code is a disposable view of it.

---

## Why this exists

For the last few months I've been running a dozen AI agents at the same time. The
bottleneck was never the code they wrote — it was my attention. Normal task
managers are built for a human clicking around; I needed one built for **assigning
work to agents and reviewing what comes back**. So I designed one.

LLMs are a fantastic tool and they still make mistakes, so every step toward a
production-grade system still needs human understanding, judgment, and review. The
win isn't automating the human away — it's redesigning the workflow so **one human
can run and review many agents in parallel**. That is the real harness, and this
app is a cockpit for it.

Two ideas turned out to matter most (both marked `novel: true` in the feature
files):

1. **Tasks are plain Markdown files.** No database. The task file *is* the prompt.
2. **A terminal runs inside the task app.** The agent works where the task lives —
   and that single decision is what unlocks the whole live-status supervision layer.

**What this repo is (and isn't).** I'm not publishing the implementation. It was
an internal tool I built for myself — I haven't read a single line of it, which is
exactly the point: internal automation and production code are not the same bar.
What I *am* publishing is the part I trust and think is reusable — the **system
design**: a feature menu, a dependency graph, and prompts you can hand to an LLM
to build your own. Every feature and every dependency edge is extracted from the
real, running app.

---

## How to use it (4 steps)

1. **Browse [`MENU.md`](MENU.md).** Features are grouped into 9 epics, at small →
   large granularity. Tick the ones you want.
2. **Resolve prerequisites.** Paste your ticked list into
   [`prompts/00-resolve.md`](prompts/00-resolve.md). It closes over the dependency
   graph in [`DEPENDENCIES.md`](DEPENDENCIES.md) and emits an ordered build plan.
   (Or start from a ready-made bundle in [`PROFILES.md`](PROFILES.md).)
3. **Build.** Hand the resolved plan + the relevant `features/*.md` files to your
   agent using [`prompts/01-build.md`](prompts/01-build.md) as the runbook.
4. **Verify.** Use [`prompts/02-verify.md`](prompts/02-verify.md) so the agent
   self-checks each feature's acceptance criteria.

`llms.txt` is the machine-readable index if you'd rather point an agent at the
whole thing and let it navigate.

---

## Relationship to Spec-Driven Development

This builds on **Spec-Driven Development** (see [GitHub Spec
Kit](https://github.com/github/spec-kit), [Amazon Kiro](https://kiro.dev), and
the "the spec is the source of truth" thesis). SDD is great at turning *one* spec
into software. This repo adds the layer SDD tools don't have: a **published,
à-la-carte feature catalog with a cross-feature dependency graph**. The resolver's
output is intentionally Spec-Kit-shaped, so if you already use `specify` you can
drop a resolved subset straight into it. Feature acceptance criteria use EARS
phrasing ("when X, the system shall Y"), same as Kiro.

The step past SDD: don't just *build* from a spec — **ship the spec**.

---

## What's NOT here

Source code. On purpose. If you want the reference app, build it from the spec.

## License

Spec text: CC BY 4.0. Build whatever you want from it.
