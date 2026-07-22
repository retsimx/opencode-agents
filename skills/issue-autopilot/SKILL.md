---
name: issue-autopilot
description: Fetch a forge issue (GitHub or GitLab), brainstorm/plan with user input, auto-implement, verify CI, open a draft PR/MR, and post a plain-English issue comment — all from an isolated git worktree.
---

# MANDATORY RULES: VIOLATION IS FORBIDDEN

- **Never skip phases.** Execute Phase 0 through Phase 6 in order. Each phase has a GATE ENTRY and GATE EXIT — both must pass before proceeding.
- **Do not modify files outside the worktree.**
- **Run ALL CI steps locally once before ANY commit or push.** Phase 4 (CI Verify) must pass before Phase 5 (Ship).
- **Provider-agnostic.** Detect GitHub (`gh`) vs GitLab (`glab`) from `origin` per `.agents/skills/_shared/runtime/providers.md`. Never hardcode one CLI.
- **Strictly follow ALL rules in the project's `AGENTS.md` and `TESTING.md`** (if they exist).
- **Phase ordering is inviolable.** Never reorder, skip, parallelize, or combine phases.
- **MUST use the `question` tool to ask the user anything.** Never use plain text output. Always invoke the built-in `question` tool.
- **MUST use the `task` tool to spawn subagents.** Never run `$ task ...` in bash — the `task` tool is a built-in agent function call, the same as `read`, `write`, or `grep`. All Task subagent delegations (Phase 5 commit messages, Phase 6 issue comment, and any exploration/fix agents) must use the `task` tool via your tool-calling interface.
- **Subagents are cheap; use them aggressively.** Spawn focused investigation or fix agents rather than doing everything inline.

---

## Phase Delegation Strategy

| Phase | Who Runs |
|:------|:---------|
| 0 (Init) | Orchestrator inline |
| 1 (Brainstorm) | Orchestrator inline |
| 2 (Plan) | Orchestrator inline |
| 3 (Implement) | Orchestrator inline |
| 4 (CI Verify) | Orchestrator inline |
| 5 (Ship) — commit msg & PR body | **Task subagent** |
| 5 (Ship) — git / forge CLI commands | Orchestrator inline |
| 6 (Issue Comment) | **Task subagent** |

---

## Phase 0: Init

### GATE ENTRY
- User has a forge issue (GitHub or GitLab) to resolve.

### Procedure

1. Ask the user for the issue number if not provided. **Use the `question` tool — NOT plain text.**
2. Detect provider from `git remote get-url origin` per `.agents/skills/_shared/runtime/providers.md`. Record `PROVIDER` (`github`|`gitlab`) and the matching CLI (`gh`|`glab`).
3. Verify the provider CLI is authenticated (`gh auth status` or `glab auth status`). If not, report error and abort.
4. Fetch issue details using the Issues table in `.agents/skills/_shared/runtime/providers.md` (normalize title, body, labels, comments, assignees, state).
5. Present the issue to the user: title, body, labels, key comments.
6. Fetch latest changes to main and ensure local main is up to date:
   ```
   git fetch origin main
   git checkout main
   git pull origin main
   ```
7. Generate a branch name from the issue title: `feat-<kebab-title>-<number>`.
8. Create a git worktree from main:
   ```
   git worktree add ../<repo-name>-<number> -b <branch> main
   ```
9. Record the worktree path as `$WORKTREE`. All subsequent phases operate from this directory. Pass `PROVIDER` into every later phase and subagent prompt.

### GATE EXIT
- [ ] Provider detected and CLI authenticated
- [ ] Issue details captured
- [ ] Local main is up to date
- [ ] Worktree created at a known path (`$WORKTREE`)
- [ ] Branch name recorded
- [ ] `PROVIDER` recorded

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

## Phase 4: CI Verify

### GATE ENTRY
- [ ] Phase 3 gate exit conditions satisfied
- [ ] Implementation is complete in `$WORKTREE`

### Procedure

1. `cd $WORKTREE`
2. Discover CI config for `$PROVIDER` per Local CI config discovery in `.agents/skills/_shared/runtime/providers.md` (GitHub: `.github/workflows/`; GitLab: `.gitlab-ci.yml`). Also follow any verify commands in `AGENTS.md`.
3. Run ALL CI steps locally inside `$WORKTREE`:
   - Install dependencies if needed.
   - Run lint, type-check, format-check, and any other static analysis steps.
   - Run the full test suite.
   - Run any coverage checks specified in the project's CI configuration.
4. If any step fails, auto-fix mechanical issues (e.g. `ruff format`, `prettier --write`) and re-run. If tests or type errors fail, report to the user with the failure output. Ask whether to re-enter Phase 3 or abort.

### GATE EXIT
- [ ] All CI steps passed (lint, typecheck, format, tests, coverage)
- [ ] Zero failing checks
- [ ] No files were modified outside `$WORKTREE`

**You CANNOT proceed to Phase 5 without satisfying ALL gate exit items.**

---

## Phase 5: Ship

**GUARD: Do not enter this phase unless Phase 4 (CI Verify) passed.**

### GATE ENTRY
- [ ] Phase 4 gate exit conditions satisfied
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
     4. Write a PR/MR body:
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

#### Part B: Execute git / forge CLI commands (ORCHESTRATOR INLINE)

4. Commit using the generated message:
   ```
   git add <specific files>    # NEVER git add -A / git add .
   git commit -F /tmp/commit-msg.txt
   ```
5. Push to the remote:
   ```
   git push -u origin <branch>
   ```
6. Create a draft PR using the Create draft PR row in `.agents/skills/_shared/runtime/providers.md` for `$PROVIDER` (title `"<type>(<scope>): <description>"`, body from `/tmp/pr-body.txt`, base/target `main`).
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

**Do NOT proceed to Phase 6 if PR creation failed. Report the error and give the user the provider-appropriate create command from `.agents/skills/_shared/runtime/providers.md`.**

---

## Phase 6: Issue Comment — TASK SUBAGENT

**If the original issue number is unknown, skip this phase and report to the user.**

### GATE ENTRY
- [ ] Phase 5 gate exit conditions satisfied
- [ ] PR created successfully
- [ ] Original issue number is known

### Procedure

**CRITICAL: Delegate this entire phase to a Task subagent using the built-in `task` tool (NOT bash). Do not write the comment inline.**

1. `cd $WORKTREE`
2. Get the branch name: `git branch --show-current`
3. Use the built-in `task` tool to write and post the issue comment:
   - `subagent_type`: `"general"`
   - `description`: `"issue-comment"`
   - `prompt`:
     ```
     Write and post a plain-English summary comment on issue #<number>.

     PROVIDER: <github|gitlab>
     WORKTREE: <$WORKTREE>
     BRANCH: <branch-name>
     SCOPE: all changes from the fork point of this branch to HEAD.

     Read .agents/skills/_shared/runtime/providers.md and use ONLY the
     Comment on issue command for PROVIDER.

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

     Fetch the issue for context using the View issue command for PROVIDER.
     Write the comment to /tmp/issue-comment.txt and post it with the
     Comment on issue command for PROVIDER.

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
| 0 | Provider CLI not authenticated | Report, abort |
| 0 | Issue not found | Report, abort |
| 0 | Worktree creation fails | Check branch conflict, prompt user |
| 1 | User rejects all approaches | Abort, clean up worktree |
| 2 | User rejects plan | Loop back to Phase 1 |
| 3 | Implementation fails repeatedly | Report error trail, ask user |
| 4 | CI fails | Report output, ask user: re-implement or abort |
| 5 | Commit/push fails | Leave worktree intact, report error |
| 5 | PR creation fails | Branch pushed, give user manual create command from `.agents/skills/_shared/runtime/providers.md` |
| 5 | Task subagent fails to generate messages | Write commit message and PR body inline as fallback |
| 6 | Issue number unknown | Skip phase, report to user |
| 6 | Comment post fails | Report error, worktree and PR are intact |

**Universal rule**: If the workflow errors after Phase 0, leave the worktree intact. Only clean up on successful Phase 6 completion or explicit user instruction.

---

## Phase Transition Map

```
Phase 0 (Init) → Phase 1 (Brainstorm) → Phase 2 (Plan) → Phase 3 (Implement)
                                                                       |
                                                                       v
                                                              Phase 4 (CI Verify) [local, 1x]
                                                                       |
                                                         ┌─────────────┤
                                                         │ FAIL        │ PASS
                                                         v             v
                                                   Phase 3 (loop)  Phase 5 (Ship) [TASK + inline]
                                                                       |
                                                                       v
                                                              Phase 6 (Comment) [TASK]
```

**TASK** = delegated to a Task subagent. Orchestrator spawns the subagent and waits for the result.
**inline** = executed directly by the orchestrator.

## References

- Provider CLI map: `.agents/skills/_shared/runtime/providers.md`
