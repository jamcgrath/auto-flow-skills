---
name: auto-plan-brief
description: Auto-flow (unattended) variant of plan-brief, invoked by /auto-dev-flow — the autonomous twin of dev-flow's plan-brief, free to drift from it. First-pass info-gathering for a feature — read .dev-flow/<task>/TICKET_CONTEXT.md if it exists, otherwise work straight from the task description the user gives (ticket or no ticket), reconnoitre the codebase (files likely to change, similar implementations, related tests, available test tooling, constraints), and write .dev-flow/<task>/PLAN_BRIEF.md as grounded input for /plan mode. The portable entry to the feature path — works with or without a tracker; no Jira required. Reconnaissance, not a plan — it gathers context and stops; it does not design the approach or edit code. Primarily invoked by /auto-dev-flow; the unattended counterpart of dev-flow's plan-brief.
---

# auto-plan-brief

A **first pass that gathers context** for `/plan` mode — reconnaissance, not a plan. It finds
what's relevant to touch so plan mode designs from a grounded base. It deliberately **stops at
gathering**: it does not choose the approach, make decisions, or edit anything. That is `/plan`
mode's job, behind your approval gate.

Second step of the feature path **when there's a ticket** (`auto-verify-ticket` → **`auto-plan-brief`** →
`/plan`); the **first** step when there isn't. Either way it's the portable funnel into plan mode
— no tracker required. A trivial, already-clear fix can skip straight to plan mode.

## Steps

1. **Get the starting context.** Read `.dev-flow/<task>/TICKET_CONTEXT.md` if it exists (from
   `/auto-verify-ticket`). If it doesn't, that's fine — **no ticket is a normal entry, not a fallback**:
   work from whatever the user gives (a typed task description, a GitHub issue, an AI brief). If
   even that is thin, proceed on the best ticket-grounded interpretation and record the assumption under Open Questions — never block for input (this runs unattended).

2. **Reconnoitre the codebase** — hand the searching to an `Explore` subagent. **If you have
   `TICKET_CONTEXT.md`, build on it — don't restate its Codebase Notes; without it, gather the
   conventions here.** Your job is the net-new, change-specific recon:
   - the exact **files (and line anchors)** likely to change to satisfy the intent
   - the **closest existing implementation to copy**, named with its path + lines
   - **related tests** that cover the area
   - **the test tooling the repo actually has** — inventory the frameworks + their run commands from
     `package.json` scripts/devDeps and config files (`playwright.config`, `vitest.config`,
     `jest.config`, `pytest.ini`, `go test`, …), and note **which layer each covers** (Playwright /
     Cypress → UI/e2e; Vitest / Jest / Mocha → unit/component; supertest / integration runners → API;
     DB-backed tests → data). Flag any layer this change touches that has **no** tooling — those criteria
     will be unverifiable (→ draft). This is what lets `/auto-author-acceptance-tests` write in the
     framework that *exists* (e.g. Vitest, not Jest) and `/auto-verify-build` run the *real* command,
     instead of guessing.
   - constraints **specific to this change** that aren't already in `TICKET_CONTEXT` — read them
     out of the existing code; never assume the stack.

   **Use the best tool available — don't default to grep.** Before text-searching, take stock of
   what this project exposes, including any **project-specific MCP servers** (a component/dependency
   graph, a usage indexer, an LSP). When one directly answers a recon question — what imports X, what's
   unused, a component's props/API — **prefer it**: it's faster and more accurate than grepping source.
   Fall back to grep / repomix / reading files when nothing fits. This stays tool-agnostic — it uses
   whatever happens to be installed and is a no-op in a project that exposes nothing extra.

   When hunting the closest existing implementation in a **large or unfamiliar** module, aim to get a
   complete **signature map in one pass** rather than grepping piecemeal (it catches a reusable thing
   iterative grep would miss) — via whatever the project offers: an LSP/symbol search, or a repomix-style
   packer **if one is installed**. If you use `repomix`, run it inside the subagent so the pack stays off
   the main thread, and always `--compress` with `--include` scoped to the relevant subtree — never
   unscoped or full-content (it floods context and adds nothing over reading the few files that matter).
   For a bounded module, skip the whole approach — Explore reading the real files is simpler and just as
   good.

3. **Bound it.** Note what's explicitly **out of scope**, and list **open questions** that need
   a human decision before building — **including any unresolved open forks flagged in
   `TICKET_CONTEXT.md`** (a contradicted criterion, an under-specified value). Carry these forward:
   this is the channel that gets them to the PLAN gate, so a fork dropped here is lost.

4. **Write `.dev-flow/<task>/PLAN_BRIEF.md`** (create the dir if missing) — `<task>` is the ticket
   key when there is one (reuse the subdirectory `/auto-verify-ticket` made), else a short kebab-case slug
   of the task, so each task keeps its own context:

   ```
   ## Goal
   [Single sentence: what this change achieves]

   ## Files to Modify
   [List, each with a one-line reason]

   ## Files for Context (read only)
   [Related files plan mode should read but not change]

   ## Similar Patterns in Codebase
   [The closest existing implementation(s) to copy, named with file + line anchors]

   ## Test Tooling
   [Frameworks present + run command + the layer each covers; flag any layer this change touches with NO tooling]

   ## Constraints
   [Only rules specific to THIS change and not already in TICKET_CONTEXT — reference it, don't repeat it]

   ## Out of Scope
   [What this change should not touch]

   ## Open Questions
   [Anything needing a human decision before building — including unresolved open forks carried from TICKET_CONTEXT.md]
   ```

5. **Hand off.** Confirm it's written, then: enter `/plan` mode referencing
   `.dev-flow/<task>/PLAN_BRIEF.md`.

## Guards
- **Gather, don't design.** No approach decisions, no edits — if you're tempted to pick a
  solution, that belongs in `/plan` mode. Surface the choice as an Open Question instead.
- **Don't duplicate `TICKET_CONTEXT`.** It already captured the intent, conventions, and flags —
  build on them. If `auto-plan-brief` would mostly restate it, the ticket was simple enough to skip
  straight to `/plan` mode. **One deliberate exception:** carry its **unresolved open forks** into
  Open Questions (step 3) — those must reach the PLAN gate, and this is the only channel that takes
  them there.
- Keep it a first pass: enough to ground plan mode, not an exhaustive audit.
