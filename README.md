# build-your-own-taskman

**A task manager, shipped as a spec — not as code.**

This repo is not source you clone. It's a **menu of features with a dependency
graph**. You tick the features you want, run one prompt that resolves the
prerequisites, and hand the result to an LLM (Claude Code, Cursor, Copilot,
Gemini CLI…). The agent builds *your* version of the app.

The spec is the product. Code is a disposable view of it.

---

## Why this exists

I spent a month running a dozen AI agents at once. The bottleneck was never the
code they wrote — it was my attention. Normal task managers are built for a human
clicking around; I needed one built for **pointing agents at work and reviewing
what comes back**. So I built it, and now I'm publishing it the way I think apps
should be published in the AI era: as a spec you can regenerate.

There are roughly three camps in the AI-tooling debate:

- **AI conservatives** — LLMs make too many mistakes; keep humans doing the work.
- **Harness maximalists** — with the right scaffolding the LLM does everything,
  no human needed.
- **Human-in-the-loop (this repo's camp)** — LLMs are a great tool that still make
  mistakes, so every step toward a production-grade system must be reviewed by a
  person. The leverage isn't removing the human — it's changing your *workflow* so
  one human can run and review **many agents in parallel**. That workflow is the
  real harness. This app is a cockpit for it.

Two ideas in the app are worth calling out (both marked `novel: true` in the
feature files):

1. **Tasks are plain Markdown files.** No database. The task file *is* the prompt.
2. **A terminal runs inside the task app.** The agent works where the task lives —
   and that single decision is what unlocks the whole live-status supervision layer.

This isn't a hypothetical template. Every feature and every dependency edge is
extracted from the real, running app.

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
