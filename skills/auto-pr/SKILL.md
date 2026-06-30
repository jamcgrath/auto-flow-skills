---
name: auto-pr
description: Auto-flow (unattended) variant of pr, invoked by /auto-dev-flow — the autonomous twin of dev-flow's pr, free to drift from it. Open a pull request whose body synthesises the Decision Log from the branch's commit messages, so the reviewer gets the intent without reading every commit. Primarily invoked by /auto-dev-flow; the unattended counterpart of dev-flow's pr. Detects a task key (e.g. Jira PROJ-1234) from the branch name and includes it only when present.
---

# auto-pr

Builds a PR description by **synthesising the Decision Logs** in the branch's commits — the
reviewer gets the why without reading every commit. Convention: intent lives in commits, the PR
rolls them up (see global CLAUDE.md).

## Steps

1. **Find base + commits.** Base is the repo's default branch (`main` unless told otherwise; some
   repos use `master`). Read the branch's commits:
   `git log --format='%H%n%s%n%b%n---END---' <base>..HEAD`
   No commits → stop and say so.

2. **Detect a task key from the branch name.** Extract the first `[A-Z][A-Z0-9]+-[0-9]+` match
   from the current branch (e.g. `feature/PROJ-1234-paywalls` → `PROJ-1234`). **Only use it if
   present** — many tasks have no key. Never invent one.

3. **Read the auto-flow artifacts, then synthesise the gate body.** The PR is the review gate, so its
   body is built for the reviewer's yes/no and **leads** with the decision aids. Read
   `.dev-flow/<task>/DECISIONS.md` (decisive forks + alternatives rejected) and
   `.dev-flow/<task>/VERIFICATION.md` (the verdict + test-integrity report), then write one body:

   ```markdown
   ## ⚠️ Decisions to confirm before merge
   <ranked, from DECISIONS.md — each: what was assumed · why (ticket-grounded) · alternative rejected>
   <omit this whole heading only if there were zero decisive forks>

   ## Verification
   **Verdict:** verified | couldn't-verify | draft: couldn't satisfy <criterion>
   <from VERIFICATION.md — state plainly that it is LLM judgment in fresh context, not ground truth>
   **Test integrity:** <tests added / changed / removed; surface any tamper breach LOUDLY>
   **Assumptions made:** <N decisive forks — call it out as an under-specification signal if high>

   ## Summary
   <1–2 lines: what this PR does, in plain terms>

   Refs: <KEY>            ← include this line ONLY if a key was found

   ### Decision Log
   **Intent:** <combined goal across the commits>
   **Approach:** <what was done overall>
   **Alternatives ruled out:**
   - <option> — <reason>   (only those that still matter at PR level)
   **Assumptions:** <inferred requirements a reviewer should sanity-check>
   **Trade-offs:** <what was sacrificed, still relevant at PR scope>

   ### Commits
   - <short-sha> <subject>

   ---
   *Rejecting?* Leave a one-line reason (`no — because X`) so the next run can correct, not cold-restart.
   ```
   Omit any empty Decision Log section; merge duplicates; drop decisions reversed later in the branch.
   If `DECISIONS.md` / `VERIFICATION.md` are absent (e.g. run outside auto-flow), skip those two blocks
   and fall back to Summary + Decision Log + Commits.

4. **Title:** concise imperative covering the branch's theme. Prefix with the key when found:
   `<KEY>: <title>` if a key was detected, otherwise just `<title>`.

5. **Create or preview — draft is decided by the verdict, here and nowhere else.** The PR opens
   **ready only when `VERIFICATION.md` says `verified`**; on `couldn't-verify`, a `draft:` verdict, or
   no `VERIFICATION.md`, open it `--draft`. `/auto-dev-flow` does **not** also decide this — this skill
   is the single source of ready-vs-draft, so the two can't drift.
   - **Remote exists** (`git remote` non-empty): push the branch if needed (`git push -u origin HEAD`),
     then `gh pr create --title '<title>' --body '<body>'` (add `--draft` per the verdict above).
     **Never merge, and never push to a protected branch** — the PR is the gate; the merge is the human's.
   - **No remote** (local-only repo): **preview** — print the title + body and write it to
     `PR_PREVIEW.md`. Say clearly it's a preview, not a real PR.

6. **Report** the PR URL (or the preview path) and the title.
