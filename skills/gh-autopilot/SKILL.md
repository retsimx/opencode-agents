---
name: gh-autopilot
description: Fetch a GitHub issue, brainstorm/plan with user input, auto-implement, review, verify CI, open a draft PR, and post a plain-English issue comment — all from an isolated git worktree.
---

# MANDATORY RULES: VIOLATION IS FORBIDDEN

- **Never skip phases.** Execute Phase 0 through Phase 8 in order. Each phase has a GATE ENTRY and GATE EXIT — both must pass before proceeding.
- **Do not modify files outside the worktree.**
- **Ralph Review (Phase 5) is the critical quality gate. Never skip or shortcut it.** Bypassing is an automatic FAIL.
- **Run ALL CI steps locally before ANY commit or push.** Phase 4 (CI Pre-Review) must pass once before review. Phase 6 (CI Post-Review) must pass twice in a row before Phase 7 (Ship).
- **Strictly follow ALL rules in the project's `AGENTS.md` and `TESTING.md`** (if they exist).
- **Phase ordering is inviolable.** Never reorder, skip, parallelize, or combine phases.
- **MUST use the `question` tool to ask the user anything.** Never use plain text output. Always invoke the built-in `question` tool.
- **MUST use the `task` tool to spawn subagents.** Never run `$ task ...` in bash — the `task` tool is a built-in agent function call, the same as `read`, `write`, or `grep`. All Task subagent delegations (Phase 5 review, Phase 7 commit messages, Phase 8 issue comment, and any exploration/fix agents) must use the `task` tool via your tool-calling interface.
- **Subagents are cheap; use them aggressively.** Spawn focused investigation or fix agents rather than doing everything inline.

---

## Phase Delegation Strategy

| Phase | Who Runs |
|:------|:---------|
| 0 (Init) | Orchestrator inline |
| 1 (Brainstorm) | Orchestrator inline |
| 2 (Plan) | Orchestrator inline |
| 3 (Implement) | Orchestrator inline |
| 4 (CI Pre-Review) | Orchestrator inline |
| 5 (Review) | **Task subagent** |
| 6 (CI Post-Review) | Orchestrator inline |
| 7 (Ship) — commit msg & PR body | **Task subagent** |
| 7 (Ship) — git/gh commands | Orchestrator inline |
| 8 (Issue Comment) | **Task subagent** |

---

## Phase 0: Init

### GATE ENTRY
- User has a GitHub issue to resolve.

### Procedure

1. Ask the user for the GitHub issue number if not provided. **Use the `question` tool — NOT plain text.**
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

### GATE EXIT
- [ ] `gh` is authenticated
- [ ] Issue details captured
- [ ] Local main is up to date
- [ ] Worktree created at a known path (`$WORKTREE`)
- [ ] Branch name recorded

**You CANNOT proceed to Phase 1 without satisfying ALL gate exit items.**

---

## Phase 1: Brainstorm

### GATE ENTRY
- [ ] Phase 0 gate exit conditions satisfied
- [ ] `$WORKTREE` is valid and accessible

### Procedure

1. `cd $WORKTREE` — enforce worktree isolation for all operations.
2. Present the issue to the user as context.
3. Load the **brainstorm** skill and follow it step by step. **You MUST execute every step in the brainstorm skill in order. Do NOT shortcut, combine, or skip any step.**
4. Save the approved design doc to `docs/plans/designs/<NNN>-<issue-title>.md`.

### GATE EXIT
- [ ] Design document saved at `docs/plans/designs/<NNN>-<issue-title>.md` inside `$WORKTREE`
- [ ] User explicitly approved the design (Step 5 blind review round completed, Step 6 saved)
- [ ] No files were modified outside `$WORKTREE`
- [ ] Brainstorm skill completed in full (all 7 steps)

**You CANNOT proceed to Phase 2 without satisfying ALL gate exit items.**

---

## Phase 2: Plan

### GATE ENTRY
- [ ] Phase 1 gate exit conditions satisfied
- [ ] Design document exists in `$WORKTREE`

### Procedure

1. `cd $WORKTREE`
2. Load the **plan** skill and follow it step by step. **You MUST execute every step in the plan skill in order. Do NOT shortcut, combine, or skip any step.**
3. **You MUST get user confirmation before proceeding to Phase 3.**

### GATE EXIT
- [ ] Machine-readable plan saved at `.agents/results/plan-{sessionId}.json` inside `$WORKTREE`
- [ ] Human-readable tracker saved at `docs/plans/work/{NNN}-{name}.md` inside `$WORKTREE` (for Medium/Complex plans)
- [ ] User explicitly confirmed the plan
- [ ] No files were modified outside `$WORKTREE`

**You CANNOT proceed to Phase 3 without satisfying ALL gate exit items.**

---

## Phase 3: Implement

### GATE ENTRY
- [ ] Phase 2 gate exit conditions satisfied
- [ ] Plan file exists in `$WORKTREE`

### Procedure

1. `cd $WORKTREE`
2. **Ask the user which implementation skill to use.** Use the built-in `question` tool (NOT plain text):

   Present these options and let the user choose ONE:

   | Option | Skill | Description |
   |:-------|:------|:------------|
   | A | `orchestrate` | Automated parallel agent execution with file-based progress tracking. Spawns multiple task subagents in priority tiers. Best for multi-domain work with independent parallel tasks. |
   | B | `work` | Coordinated multi-agent development with PM planning and QA review. Best for complex multi-domain projects with cross-agent dependencies. |
   | C | `ultrawork` | High-quality 5-phase development with 11 of 17 steps being review. Most rigorous option. Best when quality is the top priority. |

   **You MUST get the user's choice before proceeding. Do NOT assume or default.**

3. Load the chosen skill and follow it step by step using the saved plan from Phase 2. **You MUST execute every step in the chosen skill in order. Do NOT shortcut, combine, or skip any step.**
4. After implementation completes, verify no files were modified outside `$WORKTREE`.
5. If implementation fails after retries, report to the user and ask whether to continue or abort.

### GATE EXIT
- [ ] User chose an implementation skill (A, B, or C)
- [ ] Chosen skill completed all steps
- [ ] All tasks in the plan are DONE or BLOCKED
- [ ] No files were modified outside `$WORKTREE`
- [ ] If any tasks are BLOCKED, user has acknowledged and agreed to proceed

**You CANNOT proceed to Phase 4 without satisfying ALL gate exit items.**

---

## Phase 4: CI Verify — Pre-Review

### GATE ENTRY
- [ ] Phase 3 gate exit conditions satisfied
- [ ] Implementation is complete in `$WORKTREE`

### Procedure

1. `cd $WORKTREE`
2. Read `.github/workflows/ci.yml` from the repository root.
3. Run ALL CI steps locally inside `$WORKTREE`:
   - Install dependencies if needed.
   - Run lint, type-check, format-check, and any other static analysis steps.
   - Run the full test suite.
   - Run any coverage checks specified in the project's CI configuration.
4. If any step fails, report to the user with the failure output. Ask whether to re-enter Phase 3 or abort.

### GATE EXIT
- [ ] All CI steps passed (lint, typecheck, format, tests, coverage)
- [ ] Zero failing checks
- [ ] No files were modified outside `$WORKTREE`

**You CANNOT proceed to Phase 5 without satisfying ALL gate exit items.**

---

## Phase 5: Review — TASK SUBAGENT

**This is the critical quality gate. Never bypass or shortcut it.**

### GATE ENTRY
- [ ] Phase 4 gate exit conditions satisfied
- [ ] Implementation is complete in `$WORKTREE`

### Procedure

**CRITICAL: Delegate this entire phase to a Task subagent using the built-in `task` tool (NOT bash). Do not review code yourself. Do not read source files or run git diff.**

1. `cd $WORKTREE`
2. Use the built-in `task` tool to spawn a ralphreview subagent:
   - `subagent_type`: `"general"`
   - `description`: `"ralphreview"`
   - `prompt`:
     ```
     Load the ralphreview skill and execute it step by step. Scope: DIFF (all uncommitted changes in this worktree).

     IMPORTANT: Follow every rule in the ralphreview skill EXACTLY:
     - Use the task tool for all review and remediation
     - Loop until 3 consecutive clean review streaks
     - Track state in .ralphreview-state-<id> file
     - Return a clear summary: number of issues found, fixed, skipped, and final streak count

     If you hit the loop limit, return a quality warning with the current state.
     ```

3. Wait for the subagent to return a summary.
4. If the subagent reports loop-limit exceeded, present the quality warning to the user. Proceed only with user acknowledgment.
5. Verify no files outside `$WORKTREE` were modified.

### GATE EXIT
- [ ] Ralph Review subagent completed and returned a result
- [ ] Result shows either 3 clean streaks, or user acknowledged loop-limit warning
- [ ] No files were modified outside `$WORKTREE`
- [ ] All review findings are tracked in `.ralphreview-state-*` file inside `$WORKTREE`

**You CANNOT proceed to Phase 6 without satisfying ALL gate exit items.**

---

## Phase 6: CI Verify — MUST pass twice in a row

**Do NOT proceed to Phase 7 (Ship) until CI passes twice in a row.**

### GATE ENTRY
- [ ] Phase 5 gate exit conditions satisfied
- [ ] Ralph review is complete

### Procedure

1. `cd $WORKTREE`
2. Read `.github/workflows/ci.yml` from the repository root.
3. **RUN 1**: Run ALL CI steps locally inside `$WORKTREE`:
   - Install dependencies if needed.
   - Run lint, type-check, format-check, and any other static analysis steps.
   - Run the full test suite.
   - Run any coverage checks specified in the project's CI configuration.
4. If Run 1 fails: auto-fix mechanical issues (e.g. `ruff format`, `prettier --write`), then re-run from step 3. If tests or type errors fail, report to the user with the failure output. Ask whether to re-enter Phase 3 or abort.
5. **RUN 2**: Repeat step 3. Run 1 and Run 2 must both pass clean. If Run 2 fails, fix and repeat Run 1 + Run 2 until two consecutive passes are achieved.

### GATE EXIT
- [ ] Run 1 and Run 2 passed all CI steps (lint, typecheck, format, tests, coverage)
- [ ] Zero failing checks on final run
- [ ] No files were modified outside `$WORKTREE`

**You CANNOT proceed to Phase 7 without satisfying ALL gate exit items.**

---

## Phase 7: Ship

**GUARD: Do not enter this phase unless Phase 6 (CI Verify) passed.**

### GATE ENTRY
- [ ] Phase 6 gate exit conditions satisfied
- [ ] All CI checks passed
- [ ] `$WORKTREE` contains uncommitted changes

### Procedure

#### Part A: Generate commit message and PR body (TASK SUBAGENT)

**CRITICAL: Generate commit message and PR body via the built-in `task` tool (NOT bash). Do not write them inline.**

1. `cd $WORKTREE`
2. Use the built-in `task` tool to generate the commit message and PR body:
   - `subagent_type`: `"general"`
   - `description`: `"scm-commit-msg"`
   - `prompt`:
     ```
     Load the scm skill and read the Conventional Commits section. Then:

     1. Run `git diff --stat` and `git diff` to understand the changes.
     2. Determine the Conventional Commit type and scope.
     3. Write a commit message:
        - Imperative mood, lowercase, <=72 chars title, no trailing period
        - Follow the project's AGENTS.md commit style if it exists
        - Body: bullet points describing changes (no verbose numbers/filenames)
        - Reference the issue: Closes #<number>
     4. Write a PR body:
        - Bullet points describing what was done
        - Reference: Closes #<number>
     5. Write the commit message to /tmp/commit-msg.txt and the PR body to /tmp/pr-body.txt

     Return:
       COMMIT_TITLE: <the title line>
       COMMIT_FILE: /tmp/commit-msg.txt
       PR_FILE: /tmp/pr-body.txt
       TYPE: <type>
       SCOPE: <scope>
     ```

3. Wait for the subagent to return the commit title, type, scope, and file paths.

#### Part B: Execute git/gh commands (ORCHESTRATOR INLINE)

4. Commit using the generated message:
   ```
   git add <specific files>    # NEVER git add -A / git add .
   git commit -F /tmp/commit-msg.txt
   ```
5. Push to the remote:
   ```
   git push -u origin <branch>
   ```
6. Create a draft pull request:
   ```
   gh pr create --title "<type>(<scope>): <description>" --body-file /tmp/pr-body.txt --draft
   ```
7. Report the PR URL to the user.
8. Return to main branch:
   ```
   git checkout main
   ```
9. Offer to clean up the worktree:
   ```
   git worktree remove <worktree-path>
   ```

### GATE EXIT
- [ ] Commit created with Conventional Commit format
- [ ] Branch pushed to remote
- [ ] Draft PR created and URL reported to user
- [ ] Returned to main branch

**Do NOT proceed to Phase 8 if PR creation failed. Report the error and give the user the manual `gh pr create` command.**

---

## Phase 8: Issue Comment — TASK SUBAGENT

**If the original GitHub issue number is unknown, skip this phase and report to the user.**

### GATE ENTRY
- [ ] Phase 7 gate exit conditions satisfied
- [ ] PR created successfully
- [ ] Original GitHub issue number is known

### Procedure

**CRITICAL: Delegate this entire phase to a Task subagent using the built-in `task` tool (NOT bash). Do not write the comment inline.**

1. `cd $WORKTREE`
2. Get the branch name: `git branch --show-current`
3. Use the built-in `task` tool to write and post the issue comment:
   - `subagent_type`: `"general"`
   - `description`: `"issue-comment"`
   - `prompt`:
     ```
     Write and post a plain-English summary comment on GitHub issue #<number>.

     WORKTREE: <$WORKTREE>
     BRANCH: <branch-name>
     SCOPE: all changes from the fork point of this branch to HEAD.

     First, understand what was changed:
       cd <WORKTREE>
       git diff $(git merge-base main HEAD)..HEAD

     Then write a comment following these rules:
     1. NO code references — no file names, function names, line numbers, or implementation details.
     2. NO technical jargon — describe technical things in everyday language.
     3. COVER:
        - What was wrong (restate the issue).
        - Decisions made and why.
        - What was done, in plain language.
        - Side effects or behavior changes the user will notice.
     4. TONE: helpful, transparent, humble.
     5. FORMAT: short paragraphs, no headings, no bullet points.

     Fetch the issue for context:
       gh issue view <number> --json title,body

     Write the comment to /tmp/issue-comment.txt and post it:
       gh issue comment <number> --body-file /tmp/issue-comment.txt

     Return the posted comment text.
     ```

4. **Wait for the subagent to complete.** Report the posted comment to the user.

### GATE EXIT
- [ ] Comment posted on the issue
- [ ] User can see the comment content

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
| 4 | CI fails | Report output, ask user: re-implement or abort |
| 5 | Ralph exceeds loop limits | Quality warning, proceed with draft PR |
| 5 | Task subagent fails to run | Report error, ask user: skip review (risky) or abort |
| 6 | CI fails | Report output, ask user: re-implement or abort |
| 7 | Commit/push fails | Leave worktree intact, report error |
| 7 | PR creation fails | Branch pushed, give user manual `gh pr create` command |
| 7 | Task subagent fails to generate messages | Write commit message and PR body inline as fallback |
| 8 | Issue number unknown | Skip phase, report to user |
| 8 | Comment post fails | Report error, worktree and PR are intact |

**Universal rule**: If the workflow errors after Phase 0, leave the worktree intact. Only clean up on successful Phase 8 completion or explicit user instruction.

---

## Phase Transition Map

```
Phase 0 (Init) → Phase 1 (Brainstorm) → Phase 2 (Plan) → Phase 3 (Implement)
                                                                        |
                                                                        v
                                                               Phase 4 (CI Pre) [local]
                                                                        |
                                                                        v
                                                               Phase 5 (Review) [TASK]
                                                                        |
                                                                        v
                                                               Phase 6 (CI Post) [local, 2x]
                                                                        |
                                                        ┌───────────────┤
                                                        │ FAIL          │ PASS
                                                        v               v
                                                  Phase 3 (loop)  Phase 7 (Ship) [TASK + inline]
                                                                        |
                                                                        v
                                                               Phase 8 (Comment) [TASK]
```

**TASK** = delegated to a Task subagent. Orchestrator spawns the subagent and waits for the result.
**inline** = executed directly by the orchestrator.
