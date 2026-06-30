---
name: auto-verify-ticket
description: Auto-flow (unattended) variant of verify-ticket, invoked by /auto-dev-flow — the autonomous twin of dev-flow's verify-ticket, free to drift from it. Validate an externally-authored ticket, issue, or brief against the actual codebase before planning — fetch it (Jira via the Atlassian Rovo MCP, a GitHub issue via `gh`, or pasted text) plus any supplementary context files passed alongside it (handoff discovery notes, prior research), cross-reference every file/component/function/term they mention against the current repo, and write a corrected .dev-flow/<task>/TICKET_CONTEXT.md. OPTIONAL and tracker-agnostic — use it only when a description written elsewhere needs reality-checking; for an ad-hoc task you defined yourself, skip straight to /auto-plan-brief. Primarily invoked by /auto-dev-flow; the unattended counterpart of dev-flow's verify-ticket. Catches the stale file refs and renamed components that AI-written tickets are full of.
---

# auto-verify-ticket

Externally-authored work items — a Jira ticket, a GitHub issue, an AI-written brief — drift from
the code: they name files that were renamed, components that were deleted, behaviour that no
longer holds. This skill reality-checks that description against the repo you're in *before* any
planning, so plan mode starts from something true.

**Optional, conditional first step** of the feature path: **(`auto-verify-ticket`) → `auto-plan-brief` →
`/plan` mode.** Use it only when there's a description *written elsewhere* to reconcile. If you
defined the task yourself against the current code, there's nothing to verify — **skip straight
to `/auto-plan-brief`.** It answers one question — **is this description true against the code?** — and
nothing more; it doesn't design the change (that's `auto-plan-brief` + plan mode).

**It flags and flows — it is not a gate.** Drift (a renamed file, a deleted component, an
already-built or under-specified criterion) is recorded as a **Flag** and the flow continues; decisive
forks are decided later at the **PLAN gate**, not here. The one thing that stops the flow is a
**confabulation** — a ticket premised on something the repo *isn't* (step 5): *a few wrong file names
is drift; a false premise is a wrong ticket.* It can also fold in **supplementary context files**
handed alongside the ticket (handoff notes, prior research) — same reality-check, with the **code as
the arbiter** (steps 1, 4–6 and Guards).

## Steps

1. **Get the work item.** Take whatever the user has: a Jira key (`[A-Z][A-Z0-9]+-[0-9]+`), a
   GitHub issue number/URL, or pasted ticket/brief text. If they have none — or they defined the
   task themselves — this skill doesn't apply; point them to `/auto-plan-brief`. Also take any
   **supplementary context files** passed alongside it (paths to handoff notes or prior research) —
   optional, zero or more.

2. **Fetch it** from whichever source it is:
   - **Jira** → the **Atlassian Rovo MCP** (`getJiraIssue` for title/description/acceptance/labels;
     `lookupJiraAccountId` / `searchJiraIssuesUsingJql` only to resolve the key or related issues).
   - **GitHub issue** → `gh issue view <n> --json title,body,labels` (accepts a number or URL).
   - **Pasted text** → use it directly.
   - **Supplementary files** (if any) → read each path directly.
   If the fetch tool you need (Rovo / `gh`) isn't available, comment on the ticket that the required
   fetch tool is unavailable and stop — fail closed; never work around it or pause for input.

3. **Extract every concrete reference** the ticket — and any supplementary files — make: file names,
   component names, function names, routes, env vars, and domain terminology.

4. **Cross-reference each against the current repo** with file search + grep. Hand the broad
   searching to an `Explore` subagent so it stays off the main thread. Classify each reference:
   - **current** — exists as named, unchanged
   - **renamed** — exists under a different name (find and record the new one)
   - **deleted** — gone, nothing equivalent (flag)
   - **not found** — never existed / ambiguous (flag)

   This includes a supplementary file's own code citations — **confirm a note's `file:line` claim
   against the repo before you lean on it** (build on the review, don't trust it blind). A claim that
   doesn't check out is a flag, not a resolution.

5. **Check behaviour + terms — and the premise.** Flag any behaviour that conflicts with how the code
   actually works, and any terminology that doesn't match the repo's conventions. **Derive conventions
   from the codebase — never assume the stack.** Then test the ticket's *premise*: **"does this only
   make sense if the repo already had X — and it doesn't?"** If yes — it assumes a
   framework / service / architecture is **already there when it isn't** (a React component in a
   non-React app; nginx / Redis behaviour where neither exists) — that's a **confabulation** → **stop and escalate**
   (step 7, Guards). Everything short of that is drift → flag and flow: asking to *add* the thing is a
   feature, and **within-stack misattribution** (right stack, wrong layer or file — a Workers feature
   that actually lives in the Pages layer) is drift. Only a *missing-stack* premise stops the flow.

   A supplementary file may also **resolve** one of the ticket's open questions (record it resolved
   **only if its claim checked out in step 4**) or **contradict** the ticket — a flag, never a silent
   pick.

6. **Write `.dev-flow/<task>/TICKET_CONTEXT.md`** (create the dir if missing) — `<task>` is the
   verified ticket key, so each task's context sits in its own subdirectory and a new run never
   overwrites the last:

   ```
   ## Sources
   [The ticket, plus any supplementary files reconciled — so the reader knows what fed this]

   ## Intent
   [What the ticket is actually asking for, in plain language]

   ## Acceptance Criteria
   [One criterion per line, each prefixed with a verdict and a code citation where it applies:
    [verified] … · [contradicted → revise] … · [unverifiable] …]

   ## Relevant Files
   [Files to modify or reference, with current, confirmed paths]

   ## Codebase Notes
   [Similar existing patterns found; conventions to follow]

   ## Flags
   | Ticket said | Reality | Action |
   |---|---|---|
   | <reference> | current / renamed → <new> / deleted / not found / contradicted | <what to do instead> |
   ```

   With a supplementary file, **cite the source inline** (no extra column): a note that **resolves** a
   question (and checked out in step 4) gets Action *"resolved by <source>, confirmed <citation> — not
   carried"* so `auto-plan-brief` leaves it alone; a note that **conflicts** with the ticket is flagged
   (Reality: *"ticket says … / note says …"*), never silently resolved.

7. **Hand off — don't wait.** Confirm `TICKET_CONTEXT.md` is written, summarise the Flags out loud,
   then **continue without pausing for answers**: `/auto-plan-brief`, or straight into `/plan` mode
   referencing the file. **Don't turn a Flag into a question and stop** — open forks travel forward to
   the **PLAN gate** (carried in `auto-plan-brief`'s Open Questions); run standalone, with no PLAN gate,
   they wait in `TICKET_CONTEXT.md` for you. The one exception is a **confabulation** (step 5): don't
   hand off — comment the evidence on the ticket and open no PR — fail closed; never proceed on a false premise (no human is in the loop to override) (Guards).

## Guards
- **Don't invent.** If a reference is ambiguous, record it under Flags as an **open fork to raise at
  the PLAN gate** — never guess a resolution just to make the ticket look clean, and never read
  "open" as "ask the user now."
- **Don't stop the flow — one exception.** Every mismatch is a **Flag** and the flow continues;
  decisive forks are raised at the PLAN gate, not here. **Stop only on a confabulation** (step 5) — and
  even then **write the evidence, comment it on the ticket, and open no PR**, never abort silently.
  Fail-closed: holds on the auto path too.
- **Supplementary context is evidence, not truth.** Files handed alongside the ticket are reconciled
  like it, not trusted over it: the **code is the arbiter**, a note closes a fork only when its claim
  checks out, and a note-vs-ticket conflict is a flag — never a silent override. A design suggestion a
  note makes is a fork for the PLAN gate, not licence to design here.
- Used only when there's an **externally-authored** item to reconcile (Jira / GitHub / brief).
  No such item — or a task you defined yourself against current code — → skip this skill and
  start at `/auto-plan-brief`.
- `.dev-flow/` is meant to be git-ignored globally (one-time flow setup); this skill writes only
  inside `.dev-flow/<task>/` and never touches `.git/` or a shared `.gitignore` itself.
