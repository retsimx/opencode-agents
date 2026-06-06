---
description: Fetch a GitHub issue, brainstorm/plan with user input, auto-implement, review, verify CI, open a draft PR, and post a plain-English issue comment — all from an isolated git worktree.
---

# MANDATORY RULES: VIOLATION IS FORBIDDEN

- **NEVER skip phases.** Execute from Phase 0 through Phase 7 in strict order. Each phase has a GATE ENTRY check and GATE EXIT check — both must pass before proceeding.
- **Do NOT modify files outside the worktree.** All code changes happen inside the worktree directory.
- **Ralph Review (Phase 4) is the most critical quality gate. NEVER skip or shortcut it.** It verifies the implementation is correct and not nonsense. Bypassing Ralph Review is an automatic FAIL.
- **Run ALL CI steps locally before ANY commit or push.** Phase 5 (CI Verify) MUST complete successfully before Phase 6 (Ship) begins. Never commit first and verify later.
- **Strictly follow ALL rules in the project's `AGENTS.md` and `TESTING.md` (if they exist).** These project-level rules override any general coding assumptions.
- **Phase ordering is inviolable.** Never reorder, skip, parallelize, or combine phases. Each phase depends on the previous.
- **MUST use the `question` tool to ask the user anything.** Never ask questions in plain text output. Always invoke the built-in `question` tool (not bash, not inline text). This includes: asking for the issue number, asking which workflow to use, getting confirmations, requesting clarification.
- **MUST use the `task` tool to spawn subagents.** Never run `$ task ...` in bash — the `task` tool is a built-in agent function call, the same as `read`, `write`, or `grep`. All Task subagent delegations (Phase 4 review, Phase 6 commit messages, Phase 7 issue comment, and any exploration/fix agents) must use the `task` tool via your tool-calling interface.
- **Subagents are cheap; use them aggressively.** Spawn focused investigation or fix agents rather than doing everything inline. They prevent context dilution.

---

## Phase Delegation Strategy

| Phase | Who Runs | Why |
|:------|:---------|:----|
| 0 (Init) | Orchestrator inline | User input needed (issue number), worktree creation |
| 1 (Brainstorm) | Orchestrator inline | Heavy user interaction — clarifying questions, approach selection, design approval |
| 2 (Plan) | Orchestrator inline | User must review and confirm the plan |
| 3 (Implement) | Orchestrator inline | User picks workflow (orchestrate/work/ultrawork); chosen workflow spawns its own task subagents |
| 4 (Review) | **Task subagent** | Fully autonomous review loop — no user interaction. Ralph Review is designed for task delegation |
| 5 (CI Verify) | Orchestrator inline | CI runs in worktree; requires local bash access |
| 6 (Ship) — commit msg & PR body | **Task subagent** | Content generation only — no side effects. Reads diff, follows scm format |
| 6 (Ship) — git/gh commands | Orchestrator inline | Must run in worktree with local git state |
| 7 (Issue Comment) | **Task subagent** | Fully autonomous — reads issue context, writes plain-English comment, posts it |

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
3. Read and follow `.agents/workflows/brainstorm.md` step by step. **You MUST execute every step in brainstorm.md in order. Do NOT shortcut, combine, or skip any step.**
4. Save the approved design doc to `docs/plans/designs/<NNN>-<issue-title>.md`.

### GATE EXIT
- [ ] Design document saved at `docs/plans/designs/<NNN>-<issue-title>.md` inside `$WORKTREE`
- [ ] User explicitly approved the design (Step 5 blind review round completed, Step 6 saved)
- [ ] No files were modified outside `$WORKTREE`
- [ ] Brainstorm workflow completed in full (all 7 steps)

**You CANNOT proceed to Phase 2 without satisfying ALL gate exit items.**

---

## Phase 2: Plan

### GATE ENTRY
- [ ] Phase 1 gate exit conditions satisfied
- [ ] Design document exists in `$WORKTREE`

### Procedure

1. `cd $WORKTREE`
2. Read and follow `.agents/workflows/plan.md` step by step. **You MUST execute every step in plan.md in order. Do NOT shortcut, combine, or skip any step.**
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
2. **Ask the user which implementation workflow to use.** Use the built-in `question` tool (NOT plain text):

   Present these options and let the user choose ONE:

   | Option | Workflow | Description |
   |:-------|:---------|:------------|
   | A | `orchestrate` (.agents/workflows/orchestrate.md) | Automated parallel agent execution with file-based progress tracking. Spawns multiple task subagents in priority tiers. Best for multi-domain work with independent parallel tasks. |
   | B | `work` (.agents/workflows/work.md) | Coordinated multi-agent development with PM planning and QA review. Best for complex multi-domain projects with cross-agent dependencies. |
   | C | `ultrawork` (.agents/workflows/ultrawork.md) | High-quality 5-phase development with 11 of 17 steps being review. Most rigorous option. Best when quality is the top priority. |

   **You MUST get the user's choice before proceeding. Do NOT assume or default.**

3. Read and follow the chosen workflow step by step using the saved plan from Phase 2. **You MUST execute every step in the chosen workflow in order. Do NOT shortcut, combine, or skip any step.**
4. After implementation completes, verify no files were modified outside `$WORKTREE`.
5. If implementation fails after retries, report to the user and ask whether to continue or abort.

### GATE EXIT
- [ ] User chose an implementation workflow (A, B, or C)
- [ ] Chosen workflow completed all steps
- [ ] All tasks in the plan are DONE or BLOCKED
- [ ] No files were modified outside `$WORKTREE`
- [ ] If any tasks are BLOCKED, user has acknowledged and agreed to proceed

**You CANNOT proceed to Phase 4 without satisfying ALL gate exit items.**

---

## Phase 4: Review — TASK SUBAGENT ⚠️ MANDATORY — DO NOT SKIP

**This phase is the most important quality gate in the entire workflow. Never bypass it, never shortcut it, never "trust" that the implementation is correct without it.**

### GATE ENTRY
- [ ] Phase 3 gate exit conditions satisfied
- [ ] Implementation is complete in `$WORKTREE`

### Procedure

**CRITICAL: You MUST delegate this entire phase to a Task subagent using the built-in `task` tool (NOT bash). Do NOT review code yourself. Do NOT read source files to verify correctness. Do NOT run git diff yourself. The Task subagent handles everything.**

1. `cd $WORKTREE`
2. Use the built-in `task` tool to spawn a ralphreview subagent (invoke via your tool-calling interface, same as `read`/`write`/`grep`):
   - `subagent_type`: `"general"`
   - `description`: `"ralphreview"`
   - `prompt`: 
     ```
     Load the ralphreview skill from .agents/skills/ralphreview/SKILL.md and execute it step by step. Scope: DIFF (all uncommitted changes in this worktree).

     IMPORTANT: Follow every rule in the ralphreview skill EXACTLY:
     - Use the task tool for all review and remediation (never review or fix code yourself)
     - Loop until 3 consecutive clean review streaks
     - Track state in .ralphreview-state-<id> file
     - Return a clear summary: number of issues found, fixed, skipped, and final streak count

     If you hit the loop limit, return a quality warning with the current state.
     ```

3. **Wait for the subagent to complete. Do NOT intervene.** The subagent returns a summary.
4. If the subagent reports loop-limit exceeded, present the quality warning to the user. The PR will remain draft. Proceed only with user acknowledgment.
5. Verify no files outside `$WORKTREE` were modified during review remediation.

### GATE EXIT
- [ ] Ralph Review subagent completed and returned a result
- [ ] Result shows either 3 clean streaks, or user acknowledged loop-limit warning
- [ ] No files were modified outside `$WORKTREE`
- [ ] All review findings are tracked in `.ralphreview-state-*` file inside `$WORKTREE`

**You CANNOT proceed to Phase 5 without satisfying ALL gate exit items.**

---

## Phase 5: CI Verify — MUST pass before ANY commit or push

**Do NOT proceed to Phase 6 (Ship) until all steps in this phase pass. Running CI after committing defeats the purpose of verification and is forbidden.**

### GATE ENTRY
- [ ] Phase 4 gate exit conditions satisfied
- [ ] Ralph review is complete

### Procedure

1. `cd $WORKTREE`
2. Read `.github/workflows/ci.yml` from the repository root.
3. Run ALL CI steps locally inside `$WORKTREE`:
   - Install dependencies if needed.
   - Run lint, type-check, format-check, and any other static analysis steps.
   - Run the full test suite.
   - Run any coverage checks specified in the project's CI configuration.
4. For CI steps that require secrets or services unavailable locally:
   - Stage and commit changes temporarily.
   - Push to the remote branch.
   - Run `gh run watch` to wait for remote CI to complete.
   - If remote CI fails, uncommit, re-enter Phase 3 with CI error context.
5. If any step fails, report to the user with the failure output. Ask whether to re-enter Phase 3 or abort.

### GATE EXIT
- [ ] All local CI steps passed (lint, typecheck, format, tests, coverage)
- [ ] Any remote-only CI steps passed OR user acknowledged they are unavailable
- [ ] Zero failing checks
- [ ] No files were modified outside `$WORKTREE`

**You CANNOT proceed to Phase 6 without satisfying ALL gate exit items.** If any check fails, you MUST loop back to Phase 3 — never skip ahead.

---

## Phase 6: Ship

**⚠️ GUARD: Do NOT enter this phase unless Phase 5 (CI Verify) passed. If you have not completed Phase 5, go back immediately.**

### GATE ENTRY
- [ ] Phase 5 gate exit conditions satisfied
- [ ] All CI checks passed
- [ ] `$WORKTREE` contains uncommitted changes that need to be shipped

### Procedure

#### Part A: Generate commit message and PR body (TASK SUBAGENT)

**CRITICAL: Generate commit message and PR body via the built-in `task` tool (NOT bash). Do NOT write them inline. The subagent reads the diff and follows scm format rules.**

1. `cd $WORKTREE`
2. Use the built-in `task` tool to generate the commit message and PR body (invoke via your tool-calling interface, same as `read`/`write`/`grep`):
   - `subagent_type`: `"general"`
   - `description`: `"scm-commit-msg"`
   - `prompt`:
     ```
     Read .agents/workflows/scm.md Step 3B for commit format rules. Then:

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

3. **Wait for the subagent to return.** Record the commit title, type, scope, and file paths.

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

**Do NOT proceed to Phase 7 if PR creation failed. Report the error and give the user the manual `gh pr create` command.**

---

## Phase 7: Issue Comment — TASK SUBAGENT

**If the original GitHub issue number is unknown, skip this phase and report to the user.**

### GATE ENTRY
- [ ] Phase 6 gate exit conditions satisfied
- [ ] PR created successfully
- [ ] Original GitHub issue number is known

### Procedure

**CRITICAL: Delegate this entire phase to a Task subagent using the built-in `task` tool (NOT bash). Do NOT write the comment inline. The subagent reads issue context, crafts a plain-English comment, and posts it.**

1. Use the built-in `task` tool to write and post the issue comment (invoke via your tool-calling interface, same as `read`/`write`/`grep`):
   - `subagent_type`: `"general"`
   - `description`: `"issue-comment"`
   - `prompt`:
     ```
     Write and post a plain-English summary comment on GitHub issue #<number>.

     The comment must be written for the END USER who reported the issue — someone who may not be technically competent. Follow these rules STRICTLY:

     1. NO code references — never mention file names, function names, variable names, line numbers, or any implementation detail.
     2. NO technical jargon — avoid terms like "API", "endpoint", "database query", "regression test", "CI/CD", "decorator", "middleware", etc. Describe technical things in everyday language.
     3. COVER these topics:
        - Problem: What was wrong, from the user's perspective. Restate the issue to confirm understanding.
        - Decisions made: Why a particular approach was chosen.
        - Assumptions made: Any assumptions about the user's environment or usage that might not hold in all cases.
        - What was done: A high-level description of the fix, in plain language. What the user will notice changing.
        - Possible side effects: Any behavior changes the user should be aware of. Be honest about trade-offs.
     4. TONE: Helpful, transparent, and humble. If there are trade-offs or uncertainties, say so.
     5. FORMAT: One short paragraph per topic. No headings, no bullet points. Reads as a friendly, concise message.

     First fetch the issue to understand context:
       gh issue view <number> --json title,body

     Then write the comment to /tmp/issue-comment.txt and post it:
       gh issue comment <number> --body-file /tmp/issue-comment.txt

     Return the posted comment text.
     ```

2. **Wait for the subagent to complete.** Report the posted comment to the user.

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
| 4 | Ralph exceeds loop limits | Quality warning, proceed with draft PR |
| 4 | Task subagent fails to run | Report error, ask user: skip review (risky) or abort |
| 5 | CI fails locally | Report output, ask user: re-implement or abort |
| 5 | Remote CI fails | Report output, ask user: re-implement or abort |
| 6 | Commit/push fails | Leave worktree intact, report error |
| 6 | PR creation fails | Branch pushed, give user manual `gh pr create` command |
| 6 | Task subagent fails to generate messages | Write commit message and PR body inline as fallback |
| 7 | Issue number unknown | Skip phase, report to user |
| 7 | Comment post fails | Report error, worktree and PR are intact |

**Universal rule**: If the workflow errors after Phase 0, leave the worktree intact. Only clean up on successful Phase 7 completion or explicit user instruction.

---

## Phase Transition Map

```
Phase 0 (Init) ──→ Phase 1 (Brainstorm) ──→ Phase 2 (Plan) ──→ Phase 3 (Implement)
                                                                        │
                                                                        ▼
                                                               Phase 4 (Review) [TASK]
                                                                        │
                                                                        ▼
                                                               Phase 5 (CI Verify)
                                                                        │
                                                        ┌───────────────┤
                                                        │ FAIL          │ PASS
                                                        ▼               ▼
                                                  Phase 3 (loop)  Phase 6 (Ship) [TASK + inline]
                                                                        │
                                                                        ▼
                                                               Phase 7 (Comment) [TASK]
```

**TASK** = delegated to a Task subagent. The orchestrator spawns the subagent and waits for the result.
**inline** = executed directly by the orchestrator in the foreground.
