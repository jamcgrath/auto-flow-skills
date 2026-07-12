# auto-flow-skills

An **unattended**, ticket-to-PR dev flow for [Claude Code](https://claude.com/claude-code), packaged
as one **self-contained** installable plugin. It is the autonomous counterpart of
[dev-flow-skills](https://github.com/jamcgrath/dev-flow-skills): **ticket in → pull request out, with
no human in the loop.**

Where dev-flow is tuned to one developer's **front-end** work, auto-flow aims to handle a change at
**any layer** — UI, backend logic, API/service, data, CLI — by **matching verification to where the
change lives** (Playwright for UI, unit/integration/DB otherwise) rather than defaulting to the browser.

dev-flow keeps two human gates (PLAN approval, REVIEW before the PR). **auto-flow swaps both for
automated approvers** — it resolves ambiguities itself, verifies the result with an independent
falsifying agent, and opens a PR whose body surfaces every decision it made. The PR is the only gate:
**it never merges**, so a human's merge decision is where review happens — asynchronously, before
anything ships.

> ⚠️ **Phase 1 / experimental.** This is the *playbook*, meant to be run **locally for tuning** while
> it behaves exactly as it would unattended (fully non-interactive — it never asks a question). It is
> **not yet wired to any trigger** and has not been hardened against a real ticket stream. Phase 2 (a
> cloud trigger → fresh session per ticket) is deliberately deferred. Treat outputs as drafts to
> review, not as trusted automation. The design rationale lives in
> [`skills/auto-dev-flow/SPEC.md`](skills/auto-dev-flow/SPEC.md).

## Why a separate plugin (not a flag on dev-flow)

dev-flow's gates are deliberately human — that's the point of it for hands-on work. Removing them
isn't a setting; it changes the safety model. So auto-flow is its own plugin, and it relocates the
guarantee rather than dropping it: *every diff is still seen before it leaves the repo* — just at
**merge** instead of before push. A wrong PR is cheap (close it — the build was a probe that often
reveals the ticket itself was wrong); a human *fooled into merging* a wrong PR is not. Everything in
the flow exists to make that one yes/no well-informed.

It also **vendors its own copies** of the underlying chain (see below) rather than depending on
dev-flow-skills — so it can retune those skills for unattended mode without ever disturbing dev-flow,
the daily driver.

## The flow

```
/auto-dev-flow <ticket>            (NON-INTERACTIVE — never asks a question, local or cloud)
  → intake + readiness: clean tree? inherently-irreversible-action ticket? runaway? → tripped: report, NO pr
  → [auto-verify-ticket]   confabulation → comment on ticket, open NO pr · else flags flow forward
  → auto-plan-brief → /plan (internal, NO gate)
       → resolve every fork to the simplest ticket-grounded option;
         log each DECISIVE fork + the alternative rejected → DECISIONS.md
  → branch off default → persist PLAN.md
  → /auto-author-acceptance-tests → COMMITS the tests (= the build's base) → capture base
  → /auto-audit-tests → red-before-green adequacy at base: adequate / weak / inadequate → TEST_AUDIT.md
  → build per plan (/auto-implement-brief; must SATISFY the acceptance tests, never edit them)
  ┌─ build↔verify loop (orchestrator owns budget: 3 build attempts) ─────────────┐
  │   /auto-verify-build — FRESH subagent per pass → falsify vs criteria + review │
  │   test diff (weakened-old · vacuous-new · edited-acceptance) → VERIFICATION.md │
  │     verified → done · falsified → fix task (retry) · couldn't-verify/exhausted → DRAFT │
  └──────────────────────────────────────────────────────────────────────────────┘
  → /code-review (medium by default; ↑ to high only for complex multi-file logic)
  → /auto-pr  — reads DECISIONS.md + VERIFICATION.md; leads with ⚠️ decisions + verdict;
                opens READY only if verified, else DRAFT
  → STOP. Never merges. Merge = the (async) human gate.
```

## What's in it

| Skill | Role |
|---|---|
| `auto-dev-flow` | The unattended orchestrator — ticket → PR with no human in the loop; owns the build↔verify loop + budget |
| `auto-verify-ticket` | reconcile an externally-authored ticket against the code (confabulation → comment on ticket, no PR) |
| `auto-plan-brief` | feature recon — grounded context for `/plan` mode + a **test-tooling inventory** (frameworks/commands per layer); **recon fan-out scales to the surface** (one explorer for a bounded area, 2–3 parallel facets for a broad/unfamiliar one); never pauses |
| `auto-author-acceptance-tests` | **(new)** author acceptance tests from the criteria, independent of the build, and **commit them** as the base |
| `auto-audit-tests` | **(new)** audit those tests via red-before-green — adequate / weak / inadequate, so a vacuous test can't pass as "verified" |
| `auto-implement-brief` | build the plan the lean way — reuse survey → minimal build; decide-and-flag, never edits the acceptance tests |
| `auto-verify-build` | **(new)** the independent verifier — fresh subagent per pass, falsifies vs criteria + reviews the test diff → verdict |
| `auto-commit` | commit with a proportional Decision Log |
| `auto-pr` | assemble the gate PR body (⚠️ decisions + verdict + test-integrity) and open ready/draft per the verdict — never merges |

`/code-review` is a Claude Code built-in, used by the flow but not bundled here.

## Self-contained — vendored, not dependent

The five sub-skills above are **vendored copies** of their dev-flow-skills equivalents, carried here
under an `auto-` prefix. This is deliberate: auto-flow needs to be able to change an underlying skill
to suit unattended mode **without touching dev-flow** (the author's daily driver), and the prefix
avoids a name collision when both plugins are installed side by side. The cost is duplication and
intentional drift; the benefit is total isolation. See
[`skills/auto-dev-flow/SPEC.md`](skills/auto-dev-flow/SPEC.md) for the full rationale.

## Layout

```
auto-flow-skills/
  .claude-plugin/marketplace.json   # makes the set installable as a plugin
  skills/auto-dev-flow/
    SKILL.md                        # the operational playbook
    SPEC.md                         # the design rationale + decisions ruled out
  skills/auto-verify-ticket/SKILL.md
  skills/auto-plan-brief/SKILL.md
  skills/auto-author-acceptance-tests/SKILL.md
  skills/auto-audit-tests/SKILL.md
  skills/auto-implement-brief/SKILL.md
  skills/auto-verify-build/SKILL.md
  skills/auto-commit/SKILL.md
  skills/auto-pr/SKILL.md
```

## Install

```sh
/plugin marketplace add jamcgrath/auto-flow-skills
/plugin install auto-flow@auto-flow-skills
```

Then `/auto-dev-flow <ticket>`. No other plugin is required — the chain is bundled.

### For development (live edits)

Symlink the skill folders into your user skills dir so edits in this repo are live immediately:

```sh
for d in auto-dev-flow auto-verify-ticket auto-plan-brief auto-author-acceptance-tests auto-audit-tests auto-implement-brief auto-verify-build auto-commit auto-pr; do
  ln -s "$PWD/skills/$d" ~/.claude/skills/"$d"
done
```

## Things to know before you point this at anything

- **It never merges.** The PR is the gate; merge is the only human judgment and is **not** swappable,
  even in cloud. Never run this against a branch with auto-merge enabled.
- **It is fully non-interactive — even locally.** No questions, no pauses, in Phase 1 *or* Phase 2.
  Local invocation changes only the trigger, never the behaviour — otherwise you're tuning a different
  skill than the one that ships.
- **A ticket is the usual input, but not required.** Args can be a Jira key, a GitHub issue, or plain
  pasted text describing the task. With no external ticket, `auto-verify-ticket` is skipped and no
  `TICKET_CONTEXT.md` is written — so the acceptance-test, audit, and verify skills take their criteria
  from the approved plan (`PLAN.md`) or the task description instead, keeping an ad-hoc run fully
  grounded rather than leaving them with no criteria source.
- **It refuses sparingly** — by *reporting* it (commenting on the ticket when Jira write scope exists,
  else surfacing) and opening **no** PR — on: a **confabulation**, a **not-ready working tree** (dirty /
  unrelated branch), a ticket that **inherently needs an irreversible action** (a prod deploy/migration),
  or a **runaway**. **No-fly is an *action* guard, not a topic ban** — any topic may be built because the
  PR (never a deploy) is the gate; the guard fires at build/verify on actions whose effects outlive a
  closed PR (deploy, shared-DB migration, prod-mutating call, a committed secret). And **already-built is
  a valid terminal** — the flow reports "already implemented", it doesn't fabricate a PR. Everything else
  is attempted; the human's merge decision is the competence boundary.
- **Honest over confident.** A `couldn't-verify` or budget-exhausted result is a **draft** PR that says
  so — never dressed up as ready. The worst failure is a PR that falsely claims "verified," because it
  corrupts the merge decision at its source.
- **Test integrity is structural, not instructional.** You can't prompt an agent out of gaming tests.
  Defenses: separation of powers (builder ≠ verifier ≠ acceptance-test author), mechanical tamper
  tripwires in **both** directions (weakened-old *and* vacuous-new tests), and loud surfacing in the
  PR. Mutation testing — the strongest catch — is deferred. See `SPEC.md`.
- **Phase 2 (cloud) gotchas, when you get there.** Jira via the Atlassian MCP may not authenticate in
  a headless run — GitHub issues + a `gh` token are the path of least resistance for v1. Provisioning
  the environment (dev server + Playwright + a write-scoped token) is most of the setup effort.
- **`.dev-flow/` scratch dir.** The recon skills write context files under `.dev-flow/<task>/` in
  whatever repo you run them in (already in this repo's `.gitignore`). Visibility comes from
  *promoting* decisions to the PR + a ticket comment — not from committing the scratchpad.

## License

[MIT](LICENSE) © James McGrath
