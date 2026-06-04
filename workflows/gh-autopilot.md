---
description: Fetch a GitHub issue, brainstorm/plan with user input, auto-implement, review, verify CI, open a draft PR, and post a plain-English issue comment — all from an isolated git worktree.
---

# MANDATORY RULES: VIOLATION IS FORBIDDEN

- **NEVER skip phases.** Execute from Phase 0 through Phase 7 in strict order. Report completion of each phase before proceeding.
- **Do NOT modify files outside the worktree.** All code changes happen inside the worktree directory.
- **Ralph Review (Phase 4) is the most critical quality gate. NEVER skip or shortcut it.** It verifies the implementation is correct and not nonsense. Bypassing Ralph Review is an automatic FAIL.
- **Run ALL CI steps locally before ANY commit or push.** Phase 5 (CI Verify) MUST complete successfully before Phase 6 (Ship) begins. Never commit first and verify later.
- **Strictly follow ALL rules in the project's `AGENTS.md` and `TESTING.md` (if they exist).** These project-level rules override any general coding assumptions.
- **Phase ordering is inviolable.** Never reorder, skip, parallelize, or combine phases. Each phase depends on the previous.

---

## Phase 0: Init

1. Ask the user for the GitHub issue number if not provided.
2. Verify `gh` CLI is installed and authenticated (`gh auth status`). If not, report error and abort.
3. Fetch issue details:
   ```
   gh issue view <number> --json title,body,labels,comments,assignees,state
   ```
4. Present the issue to the user: title, body, labels, key comments.
5. Fetch latest changes to main and ensure local main is up to date:
   ```
   git fetch origin main
   git checkout main
   git pull origin main
   ```
6. Generate a branch name from the issue title: `feat-<kebab-title>-<number>`.
7. Create a git worktree from main:
   ```
   git worktree add ../<repo-name>-<number> -b <branch> main
   ```
8. Record the worktree path as `$WORKTREE`. All subsequent phases operate from this directory.

**Do NOT proceed** without a valid worktree.

---

## Phase 1: Brainstorm

1. `cd $WORKTREE` — enforce worktree isolation for all operations.
2. Present the issue to the user as context.
3. Read and follow `.agents/workflows/brainstorm.md` step by step.
4. Save the approved design doc to `docs/plans/designs/<NNN>-<issue-title>.md`.
5. Report design completion to the user before proceeding.

---

## Phase 2: Plan

1. Read and follow `.agents/workflows/plan.md` step by step.
2. **You MUST get user confirmation before proceeding to Phase 3.**

---

## Phase 3: Implement

1. Read and follow `.agents/workflows/orchestrate.md` step by step using the saved plan from Phase 2.
2. After orchestration completes, verify no files were modified outside `$WORKTREE`.
3. If implementation fails after retries, report to the user and ask whether to continue or abort.

---

## Phase 4: Review ⚠️ MANDATORY — DO NOT SKIP

**This phase is the most important quality gate in the entire workflow. Never bypass it, never shortcut it, never "trust" that the implementation is correct without it.**

1. Read and follow `.agents/skills/ralphreview/SKILL.md` step by step inside `$WORKTREE`.
2. If Ralph exceeds loop limits, report the quality warning to the user and proceed (PR will be draft).
3. Verify no files outside `$WORKTREE` were modified during review remediation.
4. **Do NOT proceed to Phase 5 until Ralph Review has completed** (either 3 clean streaks, or user-acknowledged loop-limit warning).

---

## Phase 5: CI Verify — MUST pass before ANY commit or push

**Do NOT proceed to Phase 6 (Ship) until all steps in this phase pass. Running CI after committing defeats the purpose of verification and is forbidden.**

1. Read `.github/workflows/ci.yml` from the repository root.
2. Run ALL CI steps locally inside `$WORKTREE`:
   - Install dependencies if needed.
   - Run lint, type-check, format-check, and any other static analysis steps.
   - Run the full test suite.
   - Run any coverage checks specified in the project's CI configuration.
3. For CI steps that require secrets or services unavailable locally:
   - Stage and commit changes temporarily.
   - Push to the remote branch.
   - Run `gh run watch` to wait for remote CI to complete.
   - If remote CI fails, uncommit, re-enter Phase 3 with CI error context.
4. If any step fails, report to the user with the failure output. Ask whether to re-enter Phase 3 or abort.
5. **Only after ALL checks pass may you proceed to Phase 6.**

---

## Phase 6: Ship

**⚠️ GUARD: Do NOT enter this phase unless Phase 5 (CI Verify) passed. If you have not completed Phase 5, go back immediately.**

1. Read and follow `.agents/workflows/scm.md` Step 3B for commit execution:
   - Determine Conventional Commit type and scope.
   - Write description (imperative, lowercase, <=72 chars, no trailing period).
   - Follow the project's `AGENTS.md` commit style rules (if they exist): short title + bullet points, no verbose numbers/filenames.
   - Reference the issue at the end: `Closes #<number>` (one per line for multiple issues).
   - Write the commit message to a temp file, then commit with `git commit -F <message-file>`.
2. Push to the remote:
   ```
   git push -u origin <branch>
   ```
3. Write the PR body to a temp file and create a draft pull request:
   ```
   cat > /tmp/pr-body.txt <<'EOF'
   - <bullet point>
   - <bullet point>

   Closes #<number>
   EOF
   gh pr create --title "<type>(<scope>): <description>" --body-file /tmp/pr-body.txt --draft
   ```
4. Report the PR URL to the user.
5. Return to main branch:
   ```
   git checkout main
   ```
6. Offer to clean up the worktree:
   ```
   git worktree remove <worktree-path>
   ```

---

## Phase 7: Issue Comment

**If the original GitHub issue number is known, post a plain-English summary comment on the issue. If the issue number is unknown, skip this phase and report to the user.**

The comment must be written for the **end user who reported the issue** — someone who may not be technically competent. Follow these rules:

1. **No code references.** Never mention file names, function names, variable names, line numbers, or any implementation detail.
2. **No technical jargon.** Avoid terms like "API", "endpoint", "database query", "regression test", "CI/CD", "decorator", "middleware", etc. If you must reference something technical, describe it in everyday language (e.g., "the way the system processes requests" instead of "the API endpoint").
3. **Cover these topics:**
   - **Problem** — What was wrong, from the user's perspective. Restate the issue to confirm understanding.
   - **Decisions made** — Why a particular approach was chosen (e.g., "we decided to handle it this way because it keeps things simple").
   - **Assumptions made** — Any assumptions about the user's environment, usage patterns, or expected behaviour that might not hold in all cases.
   - **What was done** — A high-level description of the fix, in plain language. What the user will notice changing.
   - **Possible side effects** — Any regressions, behaviour changes, or other impacts the user should be aware of. Be honest if something could behave differently now.

4. **Tone**: Helpful, transparent, and humble. If there are trade-offs or things we aren't sure about, say so.

5. **Format**: One short paragraph per topic. No headings, no bullet points. The comment should read as a friendly, concise message.

Post the comment:
```
gh issue comment <number> --body "<comment>"
```

---

## Failure Handling

| Phase | Failure | Recovery |
|-------|---------|----------|
| 0 | `gh` not authenticated | Report, abort |
| 0 | Issue not found | Report, abort |
| 0 | Worktree creation fails | Check branch conflict, prompt user |
| 1 | User rejects all approaches | Abort, clean up worktree |
| 2 | User rejects plan | Loop back to Phase 1 |
| 3 | Implementation fails repeatedly | Report error trail, ask user |
| 4 | Ralph exceeds loop limits | Quality warning, proceed with draft PR |
| 5 | CI fails locally | Report output, ask user: re-implement or abort |
| 5 | Remote CI fails | Report output, ask user: re-implement or abort |
| 6 | Commit/push fails | Leave worktree intact, report error |
| 6 | PR creation fails | Branch pushed, give user manual `gh pr create` command |
| 7 | Issue number unknown | Skip phase, report to user |
| 7 | Comment post fails | Report error, worktree and PR are intact |

**Universal rule**: If the workflow errors after Phase 0, leave the worktree intact. Only clean up on successful Phase 7 completion or explicit user instruction.
