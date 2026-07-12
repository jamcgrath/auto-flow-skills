---
name: auto-verify-build
description: Auto-flow skill — the independent build verifier. Spawned as a FRESH subagent with no builder context, given only {diff since base, acceptance criteria, acceptance-test paths}, it tries to FALSIFY the change against the criteria through each criterion's layer harness (Playwright for UI, unit/integration/DB otherwise) + the full suite, and adversarially reviews the test diff for tampering, then writes a structured verdict (verified / couldn't-verify / falsified) to .dev-flow/<task>/VERIFICATION.md. ONE pass only — /auto-dev-flow owns the retry loop and budget. Fail-closed: if it can't verify, the verdict is couldn't-verify, never a false verified. Invoked by /auto-dev-flow after each build attempt.
---

# auto-verify-build

The gate's evidence. The builder does **not** certify its own work — this skill does, from the
outside. Its honesty *is* the safety property: a false `verified` corrupts the human's merge decision
at its source, which is the one thing auto-flow can't allow.

**Spawned fresh every pass — this is non-negotiable.** Run as an **isolated subagent with NO builder
session state**, given only the diff, the acceptance criteria, and the acceptance-test paths.
`/auto-dev-flow` owns the loop and the 3-attempt budget and **re-spawns a new verifier each
iteration** — never persist this across retries, never reuse the builder's context. A verifier that
accumulates the builder's state is just the self-grading this skill exists to replace.

## Steps

1. **Gather the evidence (read-only).** `git diff <base>`, where `base` is the acceptance-test commit
   sha **recorded in `.dev-flow/<task>/ACCEPTANCE_TESTS.md`** (and also passed to you by the
   orchestrator) — you run fresh and cannot recompute it, so read it. The authored tests are *in* that
   base, so any builder edit to one surfaces in the diff. Also read the acceptance criteria
   (`.dev-flow/<task>/TICKET_CONTEXT.md` if it exists, else the approved `.dev-flow/<task>/PLAN.md` /
   task description), the protected acceptance-test paths (same manifest), and the
   **test-adequacy results** (`.dev-flow/<task>/TEST_AUDIT.md`) — a criterion whose test was judged
   `inadequate` is **not** verifiable by that test, no matter what it does now.

2. **Falsify against the criteria — using the harness for each criterion's layer.** Run the acceptance
   tests + the **full** project suite **using the repo's real commands** (from the Test Tooling
   inventory in `PLAN_BRIEF.md` — e.g. `pnpm test:unit` / `pnpm test:e2e`, not a guessed `npm test`).
   Reach each criterion the way its layer demands — **Playwright**
   for UI, the **unit runner** for logic, an **HTTP / contract client** for API/service, a **seeded DB**
   for data — and actively try to make it **fail** (edge/error inputs, not just the happy path).
   Playwright is one stream, **not** the default. The **criteria are the spec**, not the builder's
   claims. A change with **no new observable behaviour** (a refactor) is verified by the **full suite
   staying green** — there's no new criterion, so a missing acceptance test is expected, not a failure.

3. **Adversarially review the test diff — three tamper classes:**
   - **Weakened OLD tests** — assertion count down; `.skip` / `.only` / `xit` / `todo` added; a test
     deleted; coverage dropped on touched lines.
   - **Vacuous NEW tests** — tautologies (`expect(true)`, asserts-nothing) padding the count *up*.
   - **Edited ACCEPTANCE tests** — any hunk in `git diff <base>` touching a path listed in
     `ACCEPTANCE_TESTS.md`. The builder must *satisfy* these, not edit them. (Committing them as the
     base is what makes this a one-line check; on the hash-fallback, compare against the recorded hash.)
   Any of the three → a **flagged breach** (a breach makes the verdict `falsified`, step 4).

4. **Write the structured verdict** to `.dev-flow/<task>/VERIFICATION.md`:
   ```
   ## Verdict
   verified | couldn't-verify | falsified   — <one-line reason>

   ## Criteria
   - <criterion> → pass | fail | unverifiable   (+ adequacy from TEST_AUDIT.md + evidence)

   ## Test integrity
   - tests added / changed / removed: <counts>
   - tamper breaches: <none | the specific weakened/vacuous/edited-acceptance findings>
   ```
   **Factor adequacy first** (from `TEST_AUDIT.md`): a criterion whose test is `inadequate` does **not**
   count as verified even if it now passes — a vacuous pass is not a pass → `unverifiable` for that
   criterion. A `weak` test counts as a pass but is **flagged** (red-by-absence only). Only `adequate`
   tests give a clean pass. Then the verdict rules:
   - **verified** — every *testable* criterion passes on an **adequate** test AND no tamper breach.
   - **falsified** — any criterion fails, OR any tamper breach.
   - **couldn't-verify** — the **layer's harness can't run** (app / dev-server down, Playwright
     unavailable, DB or dependent service unreachable), or a criterion is unverifiable-by-nature
     (subjective). **Fail closed → this, never a false `verified`.**

5. **Return the verdict; the orchestrator owns the loop.** `falsified` → `/auto-dev-flow` hands the
   named failures back to the builder as a **fix task** (within the 3-attempt budget). `couldn't-verify`
   or budget-exhausted → a **draft** PR. This skill never loops, never fixes code, never edits tests.

## Guards
- **Fresh subagent, no builder context, one pass.** (Restated because it is the entire point — if this
  ever becomes a persistent in-loop call, the independence is gone.)
- **Run at a strong model — this is the safety gate; never downsized for a small diff.** The spawner may
  size other subagents to the change, but not this one: a weaker verifier doesn't fail more *honestly*, it
  misses tamper breaches and edge cases — i.e. false `verified`s, the one failure this flow can't allow.
- **Honest over confident.** Cannot verify → `couldn't-verify` → draft. Never dress an unverified
  change up as `verified`.
- **Falsify, don't confirm.** A verifier that only checks the happy path is theatre.
- **It is LLM judgment in fresh context** — better than self-grading, *not* ground truth. Subjective
  criteria are `unverifiable`, not a guess. (Mutation testing — the strong defense against vacuous
  tests — is deferred; see `auto-dev-flow/SPEC.md`.)
- **Read-only except `VERIFICATION.md`.** Never touches the source (the builder fixes, next loop) and
  never edits tests.
- **Non-interactive.** Never pause for input.
