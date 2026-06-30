# auto-flow-skills

An **unattended**, ticket-to-PR dev flow for [Claude Code](https://claude.com/claude-code), packaged
as one installable plugin. It is the autonomous sibling of
[dev-flow-skills](https://github.com/jamcgrath/dev-flow-skills): **ticket in → pull request out, with
no human in the loop.**

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

## Why a separate skill (not a flag on dev-flow)

dev-flow's gates are deliberately human — that's the point of it for hands-on work. Removing them
isn't a setting; it changes the safety model. So auto-flow is its own thing, and it relocates the
guarantee rather than dropping it: *every diff is still seen before it leaves the repo* — just at
**merge** instead of before push. A wrong PR is cheap (close it — the build was a probe that often
reveals the ticket itself was wrong); a human *fooled into merging* a wrong PR is not. Everything in
the flow exists to make that one yes/no well-informed.

## The flow

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
  │     verified → PR ready · falsified → retry · couldn't-verify/exhausted → DRAFT │
  └──────────────────────────────────────────────────────────────────────────────┘
  → /code-review (effort ∝ diff)
  → /pr  — body LEADS with ⚠️ decisions-to-confirm (ranked) + verdict + test-integrity report
  → STOP. Never merges. Merge = the (async) human gate.
```

## What's in it

| Skill | Role |
|---|---|
| `auto-dev-flow` | The unattended orchestrator — ticket → PR with no human in the loop |

It bundles only the orchestrator. The chain it runs is **delegated, unchanged**, to skills that live
in **dev-flow-skills** (see below).

## Depends on dev-flow-skills

auto-dev-flow is a thin orchestrator: it delegates to `verify-ticket`, `plan-brief`, `implement-brief`,
`commit`, and `pr` from
[dev-flow-skills](https://github.com/jamcgrath/dev-flow-skills), plus the `/code-review` built-in. The
only logic it adds is the five unattended-specific pieces: the fork-resolution policy, the independent
verifier + bounded build↔verify loop, the test-integrity defenses, the decision-legibility synthesis
into the PR, and fully non-interactive operation.

**Install dev-flow-skills alongside this** so those skills are reachable — they are not vendored here
(copies would drift from the originals).

## Layout

```
auto-flow-skills/
  .claude-plugin/marketplace.json   # makes the set installable as a plugin
  skills/auto-dev-flow/
    SKILL.md                        # the operational playbook
    SPEC.md                         # the design rationale + decisions ruled out
```

## Install

### As a plugin

```sh
/plugin marketplace add jamcgrath/dev-flow-skills   # the delegated chain
/plugin install dev-flow@dev-flow-skills

/plugin marketplace add jamcgrath/auto-flow-skills  # this plugin
/plugin install auto-flow@auto-flow-skills
```

Then `/auto-dev-flow <ticket>`.

### For development (live edits)

Symlink the skill folder into your user skills dir so edits in this repo are live immediately:

```sh
ln -s "$PWD/skills/auto-dev-flow" ~/.claude/skills/auto-dev-flow
```

## Things to know before you point this at anything

- **It never merges.** The PR is the gate; merge is the only human judgment and is **not** swappable,
  even in cloud. Never run this against a branch with auto-merge enabled.
- **It is fully non-interactive — even locally.** No questions, no pauses, in Phase 1 *or* Phase 2.
  Local invocation changes only the trigger, never the behaviour — otherwise you're tuning a different
  skill than the one that ships.
- **It refuses on three things only**, by commenting on the ticket and opening **no** PR: a **no-fly
  path** (auth / permissions / secrets / payments / migrations / deploy-install-network ops), a
  **confabulation** (caught by verify-ticket), or a **runaway**. Everything else is attempted — the
  human's merge decision is the competence boundary, not an up-front eligibility guess. **Confirm the
  no-fly list against your own risk surface before use** (`skills/auto-dev-flow/SKILL.md`, step 2).
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
- **`.dev-flow/` scratch dir.** The delegated recon skills write context files under `.dev-flow/<task>/`
  in whatever repo you run them in (already in this repo's `.gitignore`). Visibility comes from
  *promoting* decisions to the PR + a ticket comment — not from committing the scratchpad.

## License

[MIT](LICENSE) © James McGrath
