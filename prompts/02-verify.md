# Prompt 02 — Verify

Run after each feature (or each milestone). Paste with the feature file(s) just built.

---

You just built one or more features. Verify them honestly — do not mark anything
done you haven't actually exercised.

For each feature:
1. Restate its **acceptance criteria** (EARS bullets).
2. For each criterion, **drive the real behavior** (run the app / trigger the flow)
   and state the observed result — not "should work," but what actually happened.
3. Mark each criterion ✅ pass / ❌ fail / ⚠️ partial, with the evidence.
4. For failures, name the smallest fix and apply it, then re-verify.

Then a summary:
- Which features fully pass and are safe to keep.
- Any acceptance criterion still unmet.
- Any prerequisite you discovered was actually needed but wasn't in the plan
  (report it — the dependency graph may need an edge).

Remember: this app's philosophy is that a human reviews every step. Give me a
clear, skimmable report so I can review fast, not a wall of green checkmarks.
