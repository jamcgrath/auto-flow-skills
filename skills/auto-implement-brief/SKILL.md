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
   testable line. If anything is ambiguous, ask now — one batched question, not mid-build.

2. **Survey before building.** For each item, search the codebase for what to reuse: existing
   props, components, renderers, hooks, utilities, conventions. This is the step that prevents
   the "wrapper instead of the existing `linkTarget` prop" class of mistake. Hand broad
   searching to an `Explore` subagent so it stays off the main thread and can't mutate state.

3. **Show the reuse plan and pause.** Before writing code, present a short table:

   | Brief item | Reuse (existing) | New (only if needed) | Files |

   Flag anywhere you're adding a new abstraction, with the reason in one line. Wait for my OK.

4. **Implement minimally, item by item.** Smallest change that satisfies each item; reuse what
   the plan committed to. No drive-by refactors, extra flags, or redundant deriveds. Stay in scope.

5. **Verify behaviour, not just compilation.** Run the project's lint + typecheck. For
   UI/behavioural items, verify in a real browser (Playwright/Chrome) and report what you
   actually observed against the acceptance items.

6. **Report, then commit.** Summarise each brief item → done / blocked, noting what was reused
   vs newly added. Confirm git state (`git status`), then commit (grouped logically) and push if asked.

## Notes
- The reuse survey + plan-confirm in steps 2–3 *is* the point of this skill — don't collapse it
  into "I'll just start coding."
- For a large brief, build and verify in slices rather than one giant pass.
