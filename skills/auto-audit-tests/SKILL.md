---
name: auto-audit-tests
description: Auto-flow skill — audit the just-authored acceptance tests for ADEQUACY before the build, via "red before green". Because the tests are committed before the feature exists (base), a genuine test for new behaviour must FAIL at base; one that passes is vacuous. Spawned as a FRESH subagent (the author must not grade its own tests), it runs the tests at base and returns a THREE-valued per-test verdict — adequate (failed via a real assertion), weak (failed only by error/absence), inadequate (passed = vacuous). It classifies each criterion new/changed vs preservation itself (not from an author label). Detect-and-flag only: inadequate → criterion unverifiable, weak → softer verified. Writes TEST_AUDIT.md. Non-interactive.
---

# auto-audit-tests

Checks the acceptance tests are *worth running* before the build trusts them. `/auto-verify-build`
later confirms the code **passes** the tests — but "passes" means nothing if the test would pass
anyway. This closes that gap with **red-before-green**: the tests are committed before the feature
exists (`base`), so a real test for new behaviour must **fail at `base`**. A test that *passes*
pre-build is vacuous; a test that fails only because a symbol is missing is red but **unproven**.

**Spawned fresh** — the skill that authored the tests must not grade them — and **runs once**, between
the acceptance-test commit and the build. It judges the *tests*, not the code (there is no code yet),
and it **does not fix** them (detect-and-flag only).

## Steps

1. **Gather.** Acceptance criteria (`.dev-flow/<task>/TICKET_CONTEXT.md`); the acceptance-test paths +
   the `base` commit (`.dev-flow/<task>/ACCEPTANCE_TESTS.md`); and the **test-tooling inventory**
   (`.dev-flow/<task>/PLAN_BRIEF.md`) — you need it to know **how each runner reports an assertion
   failure vs an error**, which the whole audit turns on.

2. **Classify each criterion yourself — new/changed vs preservation.** Don't trust an author label
   (gameable). From the criterion's *intent*:
   - **new or changed behaviour** → its test must be **red at `base`** (the feature isn't there yet).
   - **preservation** ("X still works after the change") → it is *correctly* **green at `base`** and is
     verified by the **full regression suite** (which `/auto-verify-build` runs), **not** by a
     red-at-base test. Note it as regression-covered; do **not** flag it inadequate for being green.

3. **Run the new/changed tests at `base` and judge the failure KIND — three-valued.** Check out / diff
   against `base`, run each new-behaviour test, and inspect *how* it fails (parse the runner's output per
   the inventory — exit code alone is not enough):
   - **adequate** — failed via a real **assertion failure**: the code ran, produced a value, and the
     assertion caught it. The only *proven* test.
   - **weak** — failed **only by error / absence** (missing `data-testid` / symbol / route / import). Red,
     but it proves the test *references* something absent, not that its assertion is meaningful. (For UI
     tests this is the common case — a missing testid throws before any assertion runs.)
   - **inadequate** — **passed at `base`.** Vacuous — it doesn't exercise the new behaviour at all.

4. **Write `.dev-flow/<task>/TEST_AUDIT.md`** — the per-criterion result the rest of the flow reads:
   ```
   ## Per criterion
   - <criterion> · new · test <name> · adequate | weak | inadequate — <evidence>
   - <criterion> · preserved · regression-covered (no red-at-base test expected)

   ## Summary
   adequate: N · weak: N · inadequate: N · criteria with no adequate test: <list>
   ```

5. **Detect-and-flag — don't fix.** Hand the result forward; never edit or regenerate a test.
   - **inadequate** → the criterion has **no trustworthy test**; downstream treats it as
     **unverifiable** even if it later "passes" (a vacuous pass is not a pass).
   - **weak** → a **softer verified** — surfaced in the PR as red-by-absence-only, not assertion-proven.
   - **adequate** → trustworthy.

## Guards
- **Fresh + independent.** The author of the tests cannot audit them — that's self-grading.
- **Assertion-vs-error is the whole game.** A boolean red/green audit rubber-stamps almost every UI test
  (missing testid → error → "red"). Judge the failure *kind*; that categorization is the product of this
  skill.
- **The auditor owns new-vs-preserved.** Never accept an author's "this is preservation, skip the
  red-check" label — classify from the criterion's intent yourself.
- **Detect-and-flag only — no strengthen loop.** Don't regenerate tests to "make them red": that
  optimizes a gameable proxy (a spurious assertion bolted onto a missing symbol manufactures a *weak*
  red). Flag and move on; strengthening can come later, grounded in real runs.
- **Necessary, not sufficient.** Red-before-green (even assertion-red) proves a test *depends on* the new
  behaviour, not that its assertion is *complete*. Mutation testing is the *sufficient* check — deferred
  (see `auto-dev-flow/SPEC.md`).
- **Non-interactive.** Never pause for input.
