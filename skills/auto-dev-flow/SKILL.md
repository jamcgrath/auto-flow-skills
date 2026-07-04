---
name: auto-dev-flow
description: Run the dev flow fully unattended — ticket in, pull request out, with no human in the loop. It is dev-flow with BOTH human gates swapped for automated approvers: it resolves ambiguities itself (decide-and-flag), verifies the result with an independent falsifying agent inside a bounded build↔verify loop, and opens a PR whose body surfaces every decision it made — because the human's MERGE decision is the only gate. It never merges; the PR is the asynchronous review gate. It refuses (reporting back, opening no PR) only on a confabulation, a not-ready working tree, a ticket that inherently needs an irreversible action (a prod deploy/migration), or a runaway — any topic is otherwise fair game since the PR (never a deploy) is the gate, and it exits cleanly when the work is already done. Phase 1 runs locally for tuning but behaves identically to an unattended cloud run — fully non-interactive, it never asks a question. See SPEC.md (same dir) for the design rationale; it ships its own vendored chain (auto-verify-ticket, auto-plan-brief, auto-implement-brief, auto-commit, auto-pr), modeled on dev-flow but free to drift from it. Use when the user says "auto dev flow", "/auto-dev-flow <ticket>", or wants a ticket built to a PR without supervision.
---

# auto-dev-flow

The **unattended** sibling of `dev-flow`. dev-flow keeps two human gates (PLAN approval, the REVIEW
gate before the PR); **auto-dev-flow swaps both for automated approvers** and runs ticket → PR with no
human in the loop. It is the "for now" that dev-flow's own note anticipated lifting — see `SPEC.md`
for the full rationale and the decisions ruled out.

It **adds no new chain of its own** — it runs auto-flow's vendored copies of dev-flow's skills
(`auto-verify-ticket → auto-plan-brief → /plan → build → verify → /code-review → auto-pr`), free to
drift from dev-flow to suit auto mode. Everything new is confined to five things: the
**fork-resolution policy**, the **independent verifier** + the bounded **build↔verify loop**, the
**test-integrity defenses**, the **decision-legibility synthesis** into the PR, and **fully
non-interactive operation**.

**The PR is the gate.** It *never merges*. Merge is the only human judgment in the loop, so every step
exists to make that yes/no well-informed: an honest, legible PR beats a confident, wrong one. A wrong
PR is cheap (close it, the build was a probe); a human *fooled into merging* a wrong PR is not.

```
/auto-dev-flow <ticket>            (NON-INTERACTIVE — never asks a question, local or cloud)
  → intake + readiness: clean tree? inherently-irreversible-action ticket? runaway? → tripped: report, NO pr
  → [auto-verify-ticket]   confabulation → comment on ticket, open NO pr · else flags flow forward
  → auto-plan-brief → /plan (internal, NO gate)
       → resolve every fork to the simplest ticket-grounded option;
         log each DECISIVE fork + the alternative rejected → DECISIONS.md
  → branch off default → persist PLAN.md
  → /auto-author-acceptance-tests → COMMITS the tests (= the build's base) → capture base = HEAD
  → build per plan (/auto-implement-brief; must SATISFY the acceptance tests, never edit them)
  ┌─ build↔verify loop (orchestrator owns budget: 3 build attempts) ─────────────┐
  │   /auto-verify-build — FRESH subagent per pass, falsify vs criteria + review  │
  │   test diff (weakened-old · vacuous-new · edited-acceptance) → VERIFICATION.md │
  │     verified      → exit loop                                                  │
  │     falsified     → hand named failures back to builder as a FIX task (budget) │
  │     couldn't-verify / budget exhausted → exit loop (→ draft PR)                │
  │   diff-time action-guard: irreversible action / secret → abort, report        │
  └──────────────────────────────────────────────────────────────────────────────┘
  → /code-review (medium by default; ↑ to high only for complex multi-file logic)
  → /auto-pr  — reads DECISIONS.md + VERIFICATION.md; body LEADS with ⚠️ decisions +
                verdict + test-integrity; opens READY only if verified, else DRAFT
  → STOP. Never merges. Merge = the (async) human gate.
```

## Steps

1. **Take the ticket — non-interactively.** Use the ticket passed as args (Jira key / GitHub issue /
   pasted text). **Never call `AskUserQuestion` and never pause for input** — there is no human in the
   loop, and this holds even in Phase 1 local runs (see Guards). If nothing was passed, fail closed:
   comment that no ticket was supplied; do not invent one.

2. **Intake circuit-breakers + readiness (thin — not competence prediction).** Cheap pre-checks only;
   everything else is *attempted* and judged by the human at the PR (that is the competence boundary —
   see `SPEC.md`):
   - **Readiness — clean tree, right base.** `git status`. If the working tree has **uncommitted
     changes**, or you're on an **unrelated branch** (not the default, not this task's own branch),
     **stop and report** — an unattended run can't isolate the task on a dirty/unrelated base, and
     pre-existing changes would entangle the commit. (The build always branches off the **default** —
     step 6 — it never reuses whatever branch happens to be checked out.)
   - **No-fly is an ACTION guard, not a topic ban.** Branch isolation + the merge gate make the produced
     *code* safe, so **any topic may be built** (auth, infra, payments — the PR, never a deploy, is the
     gate). What's forbidden is *executing*, during build/verify, an action whose effects **outlive a
     closed PR**: a deploy/publish, a migration against a **shared/real** DB, a network-mutating call to
     a real external service (payments, email, prod data writes), or a global install. That bites at
     build/verify time (step 8) — the only **intake** refusal is a ticket that *inherently* can't be
     delivered without such an action (e.g. "run this migration on prod"). And never commit a **secret**
     into the branch (a pushed branch exposes it even unmerged).
   - **Runaway guard** — an iteration/time ceiling so a pathological run can't grind forever.
   Confabulation (step 3) plus the above are the only refusals; **fork count** is surfaced (step 9),
   never a refusal. Refuse by **reporting** it and opening no PR — comment on the ticket *when Jira write
   scope is available*; tokens are often **read-only**, so surfacing the report is the fallback.

3. **auto-verify-ticket (if an external item).** Run `/auto-verify-ticket` → `.dev-flow/<task>/TICKET_CONTEXT.md`.
   It flags drift and flows on; the one stop is a **confabulation** (a ticket premised on something the
   repo isn't) → **report the evidence (comment on the ticket if write scope, else surface it) and open
   no PR** (instead of dev-flow's human override prompt). Otherwise its flags ride forward as open forks
   for step 5.
   - **Already-done terminal.** If verify-ticket (or recon/plan) finds the ticket's criteria are
     **already satisfied in mainline** — there is nothing to build — **terminate honestly: report
     "already implemented" with the evidence, open NO PR.** Don't manufacture a PR from marginal residual
     to justify a run, and don't treat genuinely-optional residual the ticket marks "if available" as a
     reason to build. Already-built is a *valid outcome*, not a failure.

4. **auto-plan-brief → plan (internal, no gate).** Run `/auto-plan-brief` → `.dev-flow/<task>/PLAN_BRIEF.md`
   (its recon includes a **test-tooling inventory** — the frameworks + run commands
   `/auto-author-acceptance-tests` and `/auto-verify-build` will use, and any layer with no harness),
   then design the approach in `/plan` mode **strictly within scope**. The PLAN gate is *swapped, not
   skipped*: planning still happens; only the human *approval* of it is replaced by the fork policy +
   the verifier loop below.

5. **Resolve forks — decide-and-flag.** For every ambiguity, pick the **simplest option grounded in
   the ticket** and proceed. Classify as you go:
   - **Non-decisive** (an under-specified detail, two equivalent helpers) → resolve silently.
   - **Decisive** (a choice that changes *what gets built*) → resolve, then **append it to
     `.dev-flow/<task>/DECISIONS.md`** (one entry per decision — `/auto-implement-brief` appends here
     too during the build, so append, never overwrite) with: the fork, the option chosen, *why*
     (ticket-grounded), and the **alternative rejected**. These become the ⚠️ block in the PR (step 9).
   Keep decisive flags **few and ranked** — they are the reviewer's attention budget.

6. **Branch, persist the plan, then author + commit the acceptance tests.** Order matters:
   - **Branch off default** first — if on `main` / `master`, `git switch -c <branch>` named from `<task>`
     (carry the ticket key so `/auto-pr` detects it); already on a feature branch → use it.
   - **Persist the plan** to `.dev-flow/<task>/PLAN.md` — `/auto-author-acceptance-tests` reads it for the
     intended surface (routes, components, `data-testid`s).
   - **`/auto-author-acceptance-tests`** — writes acceptance tests from the criteria, independent of the
     implementation, and **commits them** (`Add acceptance tests for <task>`), recording the protected set
     in `.dev-flow/<task>/ACCEPTANCE_TESTS.md`.
   - **Capture `base = git rev-parse HEAD`** *after* that commit. The acceptance tests are now *in* the
     base, so any later builder edit to one surfaces in `git diff <base>` — this is what makes "satisfy,
     don't edit" enforceable, not aspirational. (Capture fails — unborn branch — → use the skill's hash
     fallback.)
   - **Audit the tests for adequacy** — run **`/auto-audit-tests`** (fresh subagent, model ∝ the change's
     complexity — a lighter tier for a small, simple test set) against `base` →
     `.dev-flow/<task>/TEST_AUDIT.md`. Because the feature doesn't exist at `base`, a real new-behaviour
     test must *fail* there; it judges each **adequate** (failed via a real assertion), **weak** (failed
     only by error/absence), or **inadequate** (passed = vacuous). An **inadequate** test means its
     criterion is **unverifiable** downstream — a vacuous test can't launder into "verified" even when it
     passes post-build. Detect-and-flag only; it doesn't fix tests.

7. **Build.** Build per the plan (`/auto-implement-brief` discipline) in logical increments,
   `/auto-commit` each clean increment right away with a proportional Decision Log. **Stay in scope.** The
   builder must **satisfy** the acceptance tests from step 6 by changing code — it **never edits** them
   (an edit is a flagged breach at verify).

8. **Build↔verify loop — the engine (this orchestrator owns the budget: 3 build attempts).** After each
   build attempt, run **`/auto-verify-build` as a fresh subagent** (no builder context) **at a strong
   model — never downsized for a small diff** (it's the safety gate; see the "size each spawn" Guard). It
   reads the on-disk artifacts itself, but **pass `base` into its spawn explicitly** — a fresh context can't
   recompute the rev (it's also recorded in `ACCEPTANCE_TESTS.md` as a backstop). It falsifies against the
   criteria **using each criterion's layer harness** (Playwright for UI, unit/integration/DB otherwise)
   **+ the full suite** and adversarially reviews the test diff (weakened-old · vacuous-new
   · edited-acceptance), then writes a structured verdict to `.dev-flow/<task>/VERIFICATION.md`. Handle it:
   - **verified** → exit the loop.
   - **falsified** → hand the **named failures** back to `/auto-implement-brief` as a **fix task** (not a
     fresh build) and re-verify — **re-spawn a NEW verifier each pass**; never reuse one, or you
     reintroduce the self-grading the independence is for. Stop at the 3-attempt budget.
   - **couldn't-verify** (app won't start / Playwright down / subjective criterion) **or budget
     exhausted** → exit; the PR becomes a **draft** (decided by `/auto-pr` from the verdict, step 9).
   Never loop forever; never emit a confident "ready" PR the verifier didn't confirm.
   **Diff-time action-guard:** before each commit, and **before any side-effecting verify/build command**
   (deploy/publish, a migration against a shared/real DB, a network-mutating prod call, a global install),
   **stop** — those effects outlive a closed PR. Contained local verification (unit / integration / local
   Playwright against a dev server) is fine — proceed. A **secret** about to be committed → stop. Breach →
   abort and report (comment on the ticket if write scope).

9. **code-review → PR (the decision artifact).** Run `/code-review` at **`medium` effort by default** — it
   sits in the **high-confidence band** (fewer, real findings), so it adds a second pair of eyes past the
   verify loop *without* the **nitpick flood** `high`/`max` produce (those switch on broader, lower-confidence
   coverage), and the human's *merge* review still backstops it. Escalate to **`high` only** for a diff with
   genuinely complex **cross-file / multi-step logic** — that deeper pass catches the one class of bug the
   verifier and a human skim both miss (the multi-file flow no one can trace in their head), so it earns its
   extra cost and noise *there*, not on routine diffs. **Never `max`/`ultra` in the unattended flow** —
   mostly more nitpicks for a full multi-agent cloud fan-out (a real rate-limit cost on top of the flow's own
   spawns: up to 3 verify passes + the audit). `/auto-commit` any fixes. Then **`/auto-pr`** — it reads `DECISIONS.md` + `VERIFICATION.md` and owns
   the gate body: it **leads** with the ⚠️ ranked decisions-to-confirm, the verification verdict (LLM
   judgment, not ground truth), the test-integrity report, and the assumption count, and ends with a
   "reject with one reason" prompt. **`/auto-pr` is the single owner of ready-vs-draft** — it opens
   `ready` only on a `verified` verdict, otherwise `--draft` (don't duplicate that decision here). **It
   never merges.**

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
- **Size each spawn to the change — by risk, not line count.** Give every fresh subagent a model tier
  (the `model` arg on the Agent/Task tool) proportional to the change's **complexity and stakes**, not its
  diff size — a small, low-risk diff → a lighter model (`sonnet`/`haiku`) for `/auto-audit-tests`; a large
  or subtle one → `opus`. There is **no per-spawn effort knob** for these (the Agent tool takes only
  `model`); the effort knob exists **only** for `/code-review`, whose effort level **fans out parallel
  subagents** and is the biggest rate-limit cost in the flow — so it **defaults to `medium`** (the
  high-confidence band, no nitpick flood) and steps up to `high` only for complex multi-file logic (step 9). **The one spawn that never scales down is
  `/auto-verify-build`: floor it at a strong model regardless of diff size.** It is the safety gate — a
  weaker verifier doesn't fail more *honestly*, it misses tamper breaches and edge cases, i.e. emits false
  `verified`s, the one thing this flow can't allow. And judge stakes, not size: a *tiny* diff touching
  auth / payments / a migration is high-stakes — downsize nothing.
- **Thin circuit-breakers, no competence prediction.** Refuse only on a confabulation, a not-ready tree,
  an inherently-irreversible-action ticket, or a runaway — by **reporting it** (commenting on the ticket
  when write scope allows) and opening no PR, never by half-building. **No-fly is an action guard
  enforced at build/verify (step 8), not a topic ban** — any topic may be built because the PR, not a
  deploy, is the gate. **Already-built → report it, don't build a pointless PR.** Everything else is
  attempted; the human's merge decision is the competence boundary.
- **Fail closed.** A missing tool (Rovo / `gh` / browser / dev server), an unreachable or
  ambiguous-answering verifier, or an un-capturable build base → **comment on the ticket and stop**;
  never fail open to a confident PR. (Phase 2: Jira via Rovo may not auth headless — prefer GitHub
  issues, or solve service-auth first.)
- **Thin orchestration.** Like dev-flow, it delegates to its own vendored copies of dev-flow's skills
  (free to drift); its own logic is confined to the five new things named above. It never reimplements
  a delegated skill's behaviour.
- **`.dev-flow/<task>/` stays git-ignored scratch.** Visibility comes from *promoting* decisions to
  the PR + a comment on the ticket (and, in Phase 2, a structured emission for the learning loop) — not
  from committing the scratchpad.
