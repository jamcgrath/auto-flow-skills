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

3. **Synthesise — don't concatenate.** From the commits' Decision Logs, write one PR body:

   ```markdown
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
   ```
   Omit any Decision Log section that's empty. Merge duplicates; drop decisions reversed later in the branch.

4. **Title:** concise imperative covering the branch's theme. Prefix with the key when found:
   `<KEY>: <title>` if a key was detected, otherwise just `<title>`.

5. **Create or preview:**
   - **Remote exists** (`git remote` non-empty): push the branch if needed (`git push -u origin HEAD`),
     then `gh pr create --title '<title>' --body '<body>'` (`--draft` if the user asked).
   - **No remote** (local-only repo): **preview** — print the title + body and write it to
     `PR_PREVIEW.md`. Say clearly it's a preview, not a real PR.

6. **Report** the PR URL (or the preview path) and the title.
