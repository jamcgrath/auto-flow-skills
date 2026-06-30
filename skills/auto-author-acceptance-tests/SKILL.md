---
name: auto-author-acceptance-tests
description: Auto-flow skill — author executable acceptance tests from a ticket's acceptance criteria, after /plan has fixed the surface but INDEPENDENT of the implementation, then COMMIT them so they become the build's base. Invoked by /auto-dev-flow between planning and building. Picks the test type per criterion's LAYER (UI → Playwright + data-testids, logic → unit, API/service → integration, data → seeded-DB) rather than defaulting to the browser — auto-flow is general-purpose, not front-end-only. The builder must SATISFY these tests, never edit them — committing them first is what makes any later edit visible in the diff. Half of auto-flow's test-integrity defense (the other half is /auto-verify-build). Non-interactive; never pauses for input.
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

1. **Read the criteria + the repo's test tooling, then classify each by layer.** Acceptance criteria
   from `.dev-flow/<task>/TICKET_CONTEXT.md`; the intended surface from `.dev-flow/<task>/PLAN.md`; and
   the **Test Tooling inventory** from `.dev-flow/<task>/PLAN_BRIEF.md` (the frameworks that actually
   exist + their run commands). auto-flow is **general-purpose** — for *each* criterion decide where the
   behaviour lives, because that picks the test type:
   - **UI / user flow** → Playwright black-box (key on user-visible behaviour + stable `data-testid`s)
   - **Pure logic / function** → unit test on inputs→outputs, edge cases, error paths
   - **API / service** → integration / contract test hitting the endpoint or calling the service
   - **Data / schema** → assertions against a seeded DB (+ migration up→down)
   - **CLI / library** → invoke and assert on output / exit code

   `data-testid`s apply **only** to the UI layer — don't force a backend criterion through a UI mould.
   Author each test in the framework the inventory says the repo **actually uses** (e.g. Vitest, not
   Jest) — never introduce a new one; a criterion whose layer has **no tooling** in the repo is
   **unverifiable (tooling gap)**, not licence to invent a harness. A change with **no new observable
   behaviour** (a pure refactor) gets **no acceptance test** — record that its verification is "the full
   suite stays green" (regression-only); don't invent a criterion.

2. **Author each test independent of the implementation, in its layer's idiom.** Key on the **contract**
   the builder must honour, never on internal structure it happens to choose (or the test fails for the
   wrong reason): UI → behaviour + `data-testid`s; logic → function signature + return values; API →
   endpoint + request/response shape; data → query results. Be **falsify-minded** — include the edge and
   error cases, not just the happy path. The tests reference contracts that don't exist yet — that's the
   bar: **the tests define the contract, the builder implements to it.**

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

   ## Contracts the builder must expose (UI data-testids / function signatures / endpoints)
   - <contract> — <what it is>
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
