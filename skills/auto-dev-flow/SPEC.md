# auto-dev-flow — design spec

The *why* behind the skill. `SKILL.md` is the operational playbook; this file records the decisions
that shaped it and the alternatives ruled out, so a future editor doesn't reverse-engineer the intent
from the steps. Keep them in sync: a behaviour change in `SKILL.md` that contradicts a decision here
should update the decision (or be rejected).

## What it is

A fully **unattended** sibling to `dev-flow`: **ticket in → pull request out, no human in the loop.**
`dev-flow` keeps two human gates — PLAN approval and the REVIEW gate before the PR. `auto-dev-flow`
swaps **both** for automated approvers. dev-flow already frames this as the intended extension:

> the auto path *is* the auto-approver this PLAN-gate checkpoint was always meant to be swappable for.
> The REVIEW gate (step 8) is **not** swappable for now — even an unattended run stops there for a
> human before the PR.

auto-dev-flow is that "for now" lifted. It reuses the same chain and the same delegated skills
(`verify-ticket → plan-brief → /plan → build → verify → code-review → /pr`) **unchanged**. Everything
new lives in five places only: the fork-resolution policy, the independent verifier, the test-integrity
defenses, the decision-legibility synthesis into the PR, and fully non-interactive operation.

## Core principle: the PR is the gate

It **never merges.** The PR is the asynchronous review gate, and **merge** is the single human
judgment in the loop. dev-flow's safety argument ("every diff is seen before it leaves the repo")
isn't lost — it's *relocated* from before-push to before-merge. Nothing is shipped until a human says
yes. Therefore every design choice below exists to serve one thing: **the quality of the human's
yes/no decision.** A wrong PR is cheap; a human *fooled into merging* a wrong PR is not.

## Settled decisions

1. **Decide-and-flag at decisive forks.** With no human to ask, the skill resolves every ambiguity
   to the simplest, ticket-grounded option and proceeds — but logs each *decisive* fork (one that
   changes what gets built) with the alternative it rejected, and surfaces it at the top of the PR.
   - *Rejected — abstain on decisive forks:* bounces too many good tickets; the human judging a
     concrete PR is more accurate than the agent predicting it can't decide.
   - *Rejected — confidence threshold:* needs a definition of "confidence" we don't have; decide-and-flag
     + a loud PR is simpler and the merge gate is the backstop.

2. **No competence-prediction intake gate — thin circuit-breakers only.** An earlier design tried to
   predict from the ticket whether the work was doable and refuse if not. **Rolled back:** the PR
   review already makes that judgment downstream *with the concrete artifact in hand*, which is
   strictly more accurate than predicting from an abstract ticket — and a predictive gate wrongly
   refuses good tickets (false negatives). The **PR-as-probe** insight clinched it: sometimes building
   the thing is how you *discover* the ticket was wrong, and that's cheap discovery, not waste. So the
   skill **always produces a PR** except for three hard refusals: a **confabulation** (caught by
   verify-ticket), a **no-fly path**, or a **runaway**. Fork *count* is surfaced to the reviewer as an
   under-specification signal — never used to refuse.

3. **Honest, legible PR.** The reviewer's decision is only as good as what it's based on. So: few,
   ranked, decision-relevant ⚠️ flags (alarm fatigue → rubber-stamping is the real enemy); the
   verifier's verdict stated plainly; a test-integrity report; and a near-free **"no, because X"**
   capture so a rejection feeds the next run instead of a cold restart.

4. **Independent verification, not self-grading.** The agent that built the change does not certify it.
   A **fresh-context verifier** (echoing dev-flow's independent plan-verifier) tries to *falsify* the
   change against the acceptance criteria. Ceiling stated honestly: it's still LLM judgment in fresh
   context — better than self-grading, **not** ground truth. Subjective criteria it can't certify →
   "couldn't verify" → draft PR.

5. **Test integrity is structural, not instructional.** You cannot prompt an agent out of test-gaming:
   when the goal is "make tests pass," weakening the test *is* a valid solution to that goal. So the
   defenses are structural — separation of powers + detection + visibility (detail below).

## The build↔verify loop (the engine)

The conversation never pinned this and it's the most important mechanic. The builder and the
independent verifier form a loop:

- Verifier returns a **structured verdict**: `verified` / `couldn't-verify` / `falsified`.
- `falsified` → hand the verdict back to the builder to fix → re-verify. **Budget: 2 retries (3
  build attempts total).** No human is present to break an infinite loop, so the budget is hard.
- **Exhaustion behaviour:** after the budget, open a **draft** PR flagged *"couldn't satisfy
  criterion X after N attempts"* — never a confident "ready" PR, never an endless loop.
- Verdict → PR state: `verified` → ready · `couldn't-verify` → draft (honest) · exhausted-`falsified`
  → draft + flag.

## Test integrity — detail

- **Acceptance tests authored separately and after the plan.** A dedicated step writes them from the
  ticket's acceptance criteria *once the plan fixes the intended surface* (so it knows the routes/
  test-ids), but **independent of the implementation**. Keyed on **user-visible behaviour + stable
  `data-testid`s the builder must honour** — not on internal selectors the builder happens to choose
  (or the test fails for the wrong reason). The builder must **satisfy** them, not edit them.
- **"Write-protection" is detection, not prevention — stated honestly.** Same filesystem, same user:
  the skill cannot *prevent* the builder editing a test file. "Protection" = the tamper tripwire — any
  builder edit to an acceptance test is a **flagged breach**, surfaced in the PR. The skill promises
  detection + visibility, not enforcement it can't deliver.
- **Tamper tripwires cover both directions.** Weakened *old* tests (assertion count down, `.skip` /
  `.only` / `xit` / `todo` added, a test file deleted, coverage dropped on touched lines) **and**
  vacuous *new* tests (tautologies like `expect(true)` that push the count *up*). The adversarial
  test-diff reviewer is pointed at both, because mutation testing — the real catch for vacuous tests —
  is deferred.
- **Black-box Playwright is a separate evidence stream.** The verifier confirms acceptance criteria
  through the running app, independent of the unit tests the builder wrote. Gamed unit tests + a
  black-box check that still fails = caught.
- **Deferred:** mutation testing (does a test fail when the code is mutated?) — the strongest defense
  against tautological tests, but expensive; not day-one.

## No-fly list — mechanism, checked twice

A path/content denylist (auth, payments, secrets, migrations, deploy/install/network-mutating ops),
checked at **two** points because build/verify side effects aren't undone by closing a PR:

- **Intake** — the ticket names a no-fly area → refuse, comment on the ticket, open no PR.
- **Diff-time** — the build wandered into a denylist path → **abort before committing** (and before any
  side-effecting verify command), comment on the ticket.

## Phase boundary

- **Phase 1 — local, for tuning.** Invoked by hand (`/auto-dev-flow <ticket>`) but **behaves
  identically to an unattended run: fully non-interactive, never asks a question.** Local invocation
  changes only the *trigger*, not the behaviour — otherwise you're tuning a different skill than the
  one that ships. Everything Phase 1 needs exists locally (Rovo works interactively, Playwright is
  available, unit tests run). De-risks the *playbook*. **Do not** solve headless auth or env
  provisioning here — that's drifting into Phase 2.
- **Phase 2 — cloud.** A trigger spawns a **fresh session per ticket** in a pre-provisioned
  environment (repo + dev server + Playwright + a write-scoped token). De-risks the *substrate*.
  Known gotchas: **Jira via the Rovo MCP may not auth headless** (GitHub issues + `gh` token are the
  path of least resistance for v1); environment provisioning is most of the setup effort.
