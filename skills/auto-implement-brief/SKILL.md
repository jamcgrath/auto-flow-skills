---
name: auto-implement-brief
description: Auto-flow (unattended) variant of implement-brief, invoked by /auto-dev-flow — the autonomous twin of dev-flow's implement-brief, free to drift from it. Implement a feature brief or MVP spec the lean way — survey existing code first, propose a reuse plan, then build minimally and verify in the browser before committing. Primarily invoked by /auto-dev-flow; the unattended counterpart of dev-flow's implement-brief. Front-loads a reuse survey to prevent over-engineering (the wrapper-instead-of-an-existing-prop class of mistake).
---

# auto-implement-brief

Turns a feature brief / MVP spec into shipped code in the order that prevents over-engineering:
**understand what already exists → commit to a reuse plan → build minimally → verify → ship.**
Don't skip straight to coding.

## Steps

1. **Parse the brief into a checklist.** Restate each feature / acceptance item as a concrete,
   testable line. If anything is ambiguous, **resolve it to the simplest ticket-grounded option and
   append the decision to `.dev-flow/<task>/DECISIONS.md`** (append, never overwrite — the orchestrator
   logs there too) — never pause to ask (this runs unattended).
   **On a fix iteration** (re-invoked by `/auto-dev-flow` with a `/auto-verify-build` verdict), don't
   re-parse from scratch — address only the named failing criteria.

2. **Survey before building.** For each item, search the codebase for what to reuse: existing
   props, components, renderers, hooks, utilities, conventions. This is the step that prevents
   the "wrapper instead of the existing `linkTarget` prop" class of mistake. Hand broad
   searching to an `Explore` subagent so it stays off the main thread and can't mutate state.

3. **Decide the reuse plan — log it, don't pause.** Before writing code, settle a short reuse plan:

   | Brief item | Reuse (existing) | New (only if needed) | Files |

   Anywhere you're adding a new abstraction, **append it to `.dev-flow/<task>/DECISIONS.md`** with the
   reason in one line (it surfaces in the PR's ⚠️ decisions block). **Do not wait for approval** — the
   human reviews at the PR, which is the gate.

4. **Implement minimally, item by item.** Smallest change that satisfies each item; reuse what
   the plan committed to. No drive-by refactors, extra flags, or redundant deriveds. Stay in scope.

5. **Pre-flight the acceptance tests — don't re-verify the feature.** Run the project's lint + typecheck,
   then run the **committed acceptance tests** (authored before the build by `/auto-author-acceptance-tests`)
   at whatever layer they live, and stop there. You must make them pass by **changing the code**, never by
   editing the tests — an edit to a protected acceptance test is a flagged breach. This is a **pre-flight,
   not a verdict**: running the tests you were handed is the contract, and catching a contract failure here
   is far cheaper than spending a whole `/auto-verify-build` pass on it — that gate is the budget-limited
   safety check, not your test-runner-of-first-resort. **Don't go beyond the committed tests** — no
   exploratory "does the whole feature look right" poking, no extra happy-path smoke, no browser walkthrough.
   That adversarial, at-the-layer verification is `/auto-verify-build`'s job (independent, fresh context);
   redoing it here only burns the orchestrator's context and is exactly the self-assessment the flow's
   separation of powers discounts to zero. (Consequence, accepted: you no longer eyeball criteria beyond the
   committed tests — including any the audit flagged weak/inadequate — but that signal was untrusted anyway
   and rides to the human at the merge gate.)

6. **Report, then commit.** Summarise each brief item → done / blocked, noting what was reused
   vs newly added. Confirm git state (`git status`), then `/auto-commit` each logical increment.
   **Don't push or open a PR** — that's `/auto-pr`'s job at the end of the flow.

## Notes
- The reuse survey in steps 2–3 *is* the point of this skill — don't collapse it into "I'll just
  start coding." In auto mode the plan-*confirm* becomes a **logged decision**, not a human gate.
- For a large brief, build and verify in slices rather than one giant pass.
