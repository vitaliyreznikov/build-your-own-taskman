# Prompt 00 — Resolve

Paste this into your agent along with your ticked features from `MENU.md` and the
adjacency list from `DEPENDENCIES.md`.

---

You are resolving a feature selection for the `build-your-own-taskman` spec.

**Inputs I'm giving you:**
1. My selected feature IDs (e.g. `A01, B03, F01, H01`).
2. The adjacency list from `DEPENDENCIES.md` (`FEATURE: prereq, prereq, …`).

**Do this:**
1. Compute the **transitive closure** of my selection over the adjacency list —
   add every prerequisite (and their prerequisites) until closed.
2. **Topologically sort** the closed set so no feature comes before a prerequisite.
   Break ties by epic order (A→I) then feature number.
3. If a cycle exists, stop and report the cycle instead of emitting a plan.
4. Output, in this exact shape:
   - **Added by dependency:** the features I didn't pick but that got pulled in.
   - **Build order:** the ordered list, grouped by epic, each line
     `ID — title` and the `features/ID-*.md` file to read for it.
   - **Milestones:** suggest 2–4 checkpoints where the app is runnable/testable
     (e.g. "after A+B: a working board").
5. Do **not** write code yet. This step only produces the plan.

Keep it terse. The plan feeds `prompts/01-build.md`.
