---
name: auto-dev-flow
description: Run the dev flow fully unattended — ticket in, pull request out, with no human in the loop. It is dev-flow with BOTH human gates swapped for automated approvers: it resolves ambiguities itself (decide-and-flag), verifies the result with an independent falsifying agent inside a bounded build↔verify loop, and opens a PR whose body surfaces every decision it made — because the human's MERGE decision is the only gate. It never merges; the PR is the asynchronous review gate. It refuses (commenting back on the ticket, opening no PR) only on a confabulation, a no-fly path (auth / payments / secrets / migrations), or a runaway. Phase 1 runs locally for tuning but behaves identically to an unattended cloud run — fully non-interactive, it never asks a question. See SPEC.md (same dir) for the design rationale and dev-flow for the underlying chain. Use when the user says "auto dev flow", "/auto-dev-flow <ticket>", or wants a ticket built to a PR without supervision.
---

# auto-dev-flow

The **unattended** sibling of `dev-flow`. dev-flow keeps two human gates (PLAN approval, the REVIEW
gate before the PR); **auto-dev-flow swaps both for automated approvers** and runs ticket → PR with no
human in the loop. It is the "for now" that dev-flow's own note anticipated lifting — see `SPEC.md`
for the full rationale and the decisions ruled out.

It **adds no new chain** — it runs dev-flow's existing skills (`verify-ticket → plan-brief → /plan →
build → verify → /code-review → /pr`) unchanged. Everything new is confined to five things: the
**fork-resolution policy**, the **independent verifier** + the bounded **build↔verify loop**, the
**test-integrity defenses**, the **decision-legibility synthesis** into the PR, and **fully
non-interactive operation**.

**The PR is the gate.** It *never merges*. Merge is the only human judgment in the loop, so every step
exists to make that yes/no well-informed: an honest, legible PR beats a confident, wrong one. A wrong
PR is cheap (close it, the build was a probe); a human *fooled into merging* a wrong PR is not.

```
/auto-dev-flow <ticket>            (NON-INTERACTIVE — never asks a question, local or cloud)
  → intake circuit-breakers: no-fly path? runaway? → if tripped: comment on ticket, open NO pr
  → [verify-ticket]   confabulation → comment on ticket, open NO pr · else flags flow forward
  → plan-brief → /plan (internal, NO gate)
       → resolve every fork to the simplest ticket-grounded option;
         log each DECISIVE fork + the alternative rejected → DECISIONS.md
  → author acceptance tests (separate step, after the plan, independent of the build)
  → branch off default → build per plan (must SATISFY the acceptance tests, not edit them)
  ┌─ build↔verify loop (budget: 3 build attempts) ──────────────────────────────┐
  │   independent verifier (fresh context) → falsify vs criteria + review test   │
  │   diff (weakened-old AND vacuous-new) → verdict                              │
  │     verified      → exit loop, PR ready                                       │
  │     falsified     → hand verdict back to builder, retry (within budget)       │
  │     couldn't-verify / budget exhausted → exit loop, PR = DRAFT + flag         │
  │   diff-time no-fly breach → abort before commit, comment on ticket            │
  └──────────────────────────────────────────────────────────────────────────────┘
  → /code-review (effort ∝ diff)
  → /pr  — body LEADS with ⚠️ decisions-to-confirm (ranked) + verdict + test-integrity report
  → STOP. Never merges. Merge = the (async) human gate.
```

## Steps

1. **Take the ticket — non-interactively.** Use the ticket passed as args (Jira key / GitHub issue /
   pasted text). **Never call `AskUserQuestion` and never pause for input** — there is no human in the
   loop, and this holds even in Phase 1 local runs (see Guards). If nothing was passed, fail closed:
   comment that no ticket was supplied; do not invent one.

2. **Intake circuit-breakers (thin — not competence prediction).** Two cheap refusals only; everything
   else is *attempted* and judged by the human at the PR (that is the competence boundary — see
   `SPEC.md`):
   - **No-fly path** — the ticket names a denylist area (auth, permissions, secrets, payments,
     migrations, deploy / install / network-mutating ops). → **Refuse: comment on the ticket, open no
     PR.** Build/verify side effects aren't undone by closing a PR, so this area never gets attempted
     unattended.
   - **Runaway guard** — set an iteration/time ceiling so a pathological run can't grind forever.
   Fork *count* is **not** a refusal — a ticket needing many assumptions still gets a PR; the count is
   surfaced to the reviewer as an under-specification signal (step 9).

3. **verify-ticket (if an external item).** Run `/verify-ticket` → `.dev-flow/<task>/TICKET_CONTEXT.md`.
   It flags drift and flows on; the one stop is a **confabulation** (a ticket premised on something the
   repo isn't). On the unattended path a confabulation → **comment the evidence on the ticket and open
   no PR** (instead of dev-flow's human override prompt). Otherwise its flags ride forward as open
   forks for step 5.

4. **plan-brief → plan (internal, no gate).** Run `/plan-brief` → `.dev-flow/<task>/PLAN_BRIEF.md`,
   then design the approach in `/plan` mode **strictly within scope**. The PLAN gate is *swapped, not
   skipped*: planning still happens; only the human *approval* of it is replaced by the fork policy +
   the verifier loop below.

5. **Resolve forks — decide-and-flag.** For every ambiguity, pick the **simplest option grounded in
   the ticket** and proceed. Classify as you go:
   - **Non-decisive** (an under-specified detail, two equivalent helpers) → resolve silently.
   - **Decisive** (a choice that changes *what gets built*) → resolve, then **log it to
     `.dev-flow/<task>/DECISIONS.md`** with: the fork, the option chosen, *why* (ticket-grounded), and
     the **alternative rejected**. These become the ⚠️ block in the PR (step 9).
   Keep decisive flags **few and ranked** — they are the reviewer's attention budget.

6. **Author the acceptance tests — separately, after the plan.** A dedicated step (a fresh role, not
   the builder) writes tests from the ticket's **acceptance criteria**, now that the plan fixes the
   intended surface. Key them on **user-visible behaviour + stable `data-testid`s the builder must
   honour** — never on internal selectors the builder will choose, or they fail for the wrong reason.
   The builder must **satisfy** these, not edit them.

7. **Branch, then build.** If on the repo's default branch, `git switch -c <branch>` named from
   `<task>` (carry the ticket key so `/pr` detects it). Persist the plan to `.dev-flow/<task>/PLAN.md`.
   Then build per the plan (`implement-brief` discipline) in logical increments, `/commit` each clean
   increment right away with a proportional Decision Log. **Stay in scope.** The builder satisfies the
   acceptance tests from step 6; it does not modify them.

8. **Build↔verify loop (the engine; budget = 3 build attempts).** Spawn an **independent verifier in
   fresh context** (no builder session state). It does two jobs and returns a **structured verdict**:
   - **Falsify** the change against the acceptance criteria — black-box via Playwright against the
     running app + the unit tests, as a separate evidence stream from the builder's own claims.
   - **Adversarially review the test diff** for tampering, **both directions**: weakened *old* tests
     (assertion count down; `.skip` / `.only` / `xit` / `todo` added; a test deleted; coverage dropped
     on touched lines) **and** vacuous *new* tests (tautologies like `expect(true)` padding the count
     up). Any builder edit to a step-6 acceptance test is a **flagged breach**.
   Verdict handling: `verified` → exit, PR ready · `falsified` → hand the verdict back to the builder,
   fix, re-verify **within budget** · `couldn't-verify` (subjective / unprovable criterion) or **budget
   exhausted** → exit, PR becomes a **draft** flagged *"couldn't satisfy <criterion> after N
   attempts."* Never loop forever; never emit a confident "ready" PR the verifier didn't confirm.
   **Diff-time no-fly check:** before each commit and before any side-effecting verify command, re-check
   the no-fly denylist against everything changed since the build's base — a breach → **abort before
   committing**, comment on the ticket.

9. **code-review → PR (the decision artifact).** Run `/code-review` at effort proportional to the diff.
   Then `/pr` — but the body is built for the reviewer's yes/no and **leads** with, in order:
   - **⚠️ Decisions to confirm** — the ranked decisive forks from `DECISIONS.md`, each with the
     alternative rejected. *Pinned at the top; never buried.*
   - **Verification verdict** — `verified` / `couldn't-verify` / `draft: couldn't satisfy X`, stated
     plainly (it is LLM judgment, not ground truth — say so).
   - **Test-integrity report** — tests added/changed/removed, any tamper flags.
   - **Under-specification signal** — how many assumptions this ticket required.
   - A lightweight **"reject with one reason"** prompt so a "no" is captured as `no — because X`
     (feeds the next run; cheaper than a cold restart). Then the synthesised Decision Log as usual.
   The PR opens as **ready** only on a `verified` verdict; otherwise **draft**. **It never merges.**

## Guards

- **Never merges — the PR is the gate.** Merge is the only human judgment in the loop and is **not**
  swappable, even in cloud. Every diff is still seen before it leaves the repo — just at merge, not
  before push. Never auto-merge, never push to a protected branch.
- **Fully non-interactive — even locally.** No `AskUserQuestion`, no pausing for input, in Phase 1
  *or* Phase 2. Local invocation changes only the trigger; the behaviour is identical, or you are
  tuning a different skill than the one that ships. The human watches and tweaks the policy — they are
  not in the loop.
- **Honest over confident.** A `couldn't-verify` or budget-exhausted result is a **draft** PR that says
  so — never dressed up as ready. The single worst failure is a PR that falsely claims "tests pass /
  verified," because it corrupts the human's yes/no at the source.
- **Test integrity is structural, not instructional.** You cannot prompt the builder out of gaming
  tests. Rely on separation of powers (builder ≠ verifier ≠ acceptance-test author), mechanical tamper
  tripwires in **both** directions, and loud surfacing — not on good intent. (See `SPEC.md`; mutation
  testing is the deferred strong defense.)
- **Thin circuit-breakers, no competence prediction.** Refuse only on a no-fly path, a confabulation,
  or a runaway — and refuse by **commenting on the ticket and opening no PR**, never by half-building.
  Everything else is attempted; the human's merge decision is the competence boundary.
- **Fail closed.** A missing tool (Rovo / `gh` / browser / dev server), an unreachable or
  ambiguous-answering verifier, or an un-capturable build base → **comment on the ticket and stop**;
  never fail open to a confident PR. (Phase 2: Jira via Rovo may not auth headless — prefer GitHub
  issues, or solve service-auth first.)
- **Thin orchestration.** Like dev-flow, it delegates to the existing skills unchanged; its own logic
  is confined to the five new things named above. It never reimplements a delegated skill's behaviour.
- **`.dev-flow/<task>/` stays git-ignored scratch.** Visibility comes from *promoting* decisions to
  the PR + a comment on the ticket (and, in Phase 2, a structured emission for the learning loop) — not
  from committing the scratchpad.
