---
name: auto-author-acceptance-tests
description: Auto-flow skill — author executable acceptance tests from a ticket's acceptance criteria, after /plan has fixed the surface but INDEPENDENT of the implementation, then COMMIT them so they become the build's base. Invoked by /auto-dev-flow between planning and building. Keys tests on user-visible behaviour + stable data-testids the builder must honour; the builder must SATISFY these tests, never edit them — committing them first is what makes any later edit visible in the diff. Half of auto-flow's test-integrity defense (the other half is /auto-verify-build). Non-interactive; never pauses for input.
---

# auto-author-acceptance-tests

Turns acceptance criteria into **executable tests the builder must satisfy**, authored *independently
of the implementation* so they can't be shaped to whatever the builder happens to write. Committing
them up front is what makes "the builder satisfies a fixed bar, it doesn't edit the bar" **enforceable**
rather than aspirational — see step 4.

Runs **between planning and building** in auto-flow: `/auto-plan-brief → /plan → **author acceptance
tests** → build → /auto-verify-build`. It writes tests, commits them, and records a manifest; it does
**not** implement the feature.

## Steps

1. **Read the criteria + the intended surface.** Acceptance criteria from
   `.dev-flow/<task>/TICKET_CONTEXT.md`; the routes / components / `data-testid`s the plan intends from
   `.dev-flow/<task>/PLAN.md`. Turn each criterion into one or more concrete, executable assertions.

2. **Author tests independent of the implementation.** Key them on **user-visible behaviour + stable
   `data-testid`s the builder must honour** — never on internal selectors or structure the builder will
   choose, or the test fails for the wrong reason. Black-box where possible: Playwright against the
   running app for behavioural criteria; unit tests for pure logic. The tests will reference
   `data-testid`s that don't exist yet — that's the contract: **the tests define the testids, the
   builder implements to them.**

3. **Mark unverifiable criteria honestly.** A subjective / aesthetic criterion that can't be expressed
   as an executable assertion → record it as **unverifiable**. Do **not** fake a tautological test
   (`expect(true)`) to make a number go up — honesty here feeds `/auto-verify-build`; a fake poisons it.

4. **Commit the tests immediately — this is the mechanism, not bookkeeping.** `git add` *only* the new
   test files and commit `Add acceptance tests for <task>` (via `/auto-commit`). This commit becomes the
   **build's base**, so any later builder edit to an acceptance test shows up trivially in
   `git diff <base>` — that one fact is what turns "must not edit the tests" from an unenforceable
   promise into a one-line check in `/auto-verify-build`. Two things to handle:
   - This commit is **intentionally red** — the tests reference `data-testid`s / behaviour the build
     hasn't added yet. If a pre-commit hook runs the suite it will reject the commit, so commit with
     `--no-verify` (it goes green once the build lands).
   - **Record the commit's sha as `base` in the manifest** (step 5) — `/auto-verify-build` runs in a
     fresh context and cannot recompute it.
   (Unborn branch / can't commit → fall back to recording each authored file's path **and content hash**
   in the manifest so edits are still detectable; commit-first is preferred.)

5. **Write the manifest** `.dev-flow/<task>/ACCEPTANCE_TESTS.md` — the **protected set**
   `/auto-verify-build` and the builder both read:
   ```
   ## Base commit
   <sha of the "Add acceptance tests" commit — /auto-verify-build diffs against this>

   ## Acceptance test files (protected — builder must satisfy, not edit)
   - <path>            (+ content hash, only on the can't-commit fallback)

   ## Criterion → test
   - <criterion> → <test name / file>  ·  or:  unverifiable — <why>

   ## data-testids the builder must expose
   - <testid> — <what it marks>
   ```

## Guards
- **Independent of the implementation; authored once, here, before the build.** The point is a *fixed*
  bar — never re-author or relax tests later to match what got built.
- **Don't fake coverage.** A criterion you can't test executably is `unverifiable`, not a tautology.
- **Commit-first is the integrity mechanism.** Skipping the commit (without the hash fallback) silently
  disables the acceptance-test tamper check downstream.
- **Fail closed.** If the criteria are too thin to derive any test, record them all as `unverifiable`
  and say so — never invent acceptance behaviour the ticket didn't state.
- **Non-interactive.** Never pause for input; this runs unattended.
