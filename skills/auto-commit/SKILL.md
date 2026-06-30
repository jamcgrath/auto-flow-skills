---
name: auto-commit
description: Auto-flow (unattended) variant of commit, invoked by /auto-dev-flow — the autonomous twin of dev-flow's commit, free to drift from it. Stage and commit the current changes with a proportional Decision Log in the body (Intent / Approach / Alternatives ruled out / Assumptions / Trade-offs), splitting unrelated changes into separate commits. Primarily invoked by /auto-dev-flow; the unattended counterpart of dev-flow's commit. It enforces the Decision Log commit convention even when ambient CLAUDE.md guidance gets missed.
---

# auto-commit

Writes commits that carry intent. The diff already records *what* changed; this skill makes
sure the *why* travels with it, in the Decision Log format (see global CLAUDE.md).

## Steps

1. **See what changed.** `git status --short` and `git diff` (and `git diff --staged` if anything
   is staged). No changes → stop and say so.

2. **Group into logical commits.** One logical change per commit. If the changes span unrelated
   concerns (e.g. a bug fix *and* an unrelated copy tweak), **split them** — commit each group
   separately rather than lumping them. Record the grouping (it shows in the commit subjects); don't pause — this runs unattended. Leave stray
   files that don't belong (build artifacts, files from another branch) untracked — don't `git add -A`.

3. **Write the Decision Log — source the *why* from context, not the diff.** The diff gives you
   `Approach` (what was done). `Intent`, `Alternatives`, `Assumptions`, and `Trade-offs` come from
   the task you were doing and the choices you made this session — that's the part a reviewer
   can't recover from the diff. Plain labels (commit bodies aren't markdown-rendered); the subject
   is a concise imperative line with **no task key needed**:

   ```
   <concise subject, imperative mood>

   Intent: what problem this solves and the goal
   Approach: what was done and the core reasoning
   Alternatives ruled out:
   - <option> — rejected because <reason>
   Assumptions: any inferred requirements or constraints
   Trade-offs: what was sacrificed for the chosen approach
   ```

   **Proportional:** omit any empty section — a trivial change is just `Intent` + `Approach`; a
   real decision earns all five. Never pad with "Assumptions: none".

4. **Commit.** Stage that group and commit with `git commit -F -` (heredoc) so the body formats
   correctly. Keep the `Co-Authored-By` trailer. Confirm git state first; never touch `.git`.

5. **Repeat** for each remaining group, then report the commits made (sha + subject).

## Guards
- **Don't invent intent.** If you genuinely don't know *why* a change was made (e.g. pre-existing
  edits you didn't author), state that plainly in the commit body rather than fabricating one — and never pause for input (this runs unattended).
- Don't dress up trivial changes to look important. Proportionality is the point.
