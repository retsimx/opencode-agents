---
name: issue-autopilot
description: Fetch a forge issue (GitHub or GitLab), brainstorm/plan with user input, auto-implement via ultrawork, verify fast-fail CI, open a draft PR/MR, and post a plain-English issue comment — all from an isolated git worktree.
---

# MANDATORY RULES: VIOLATION IS FORBIDDEN

- **Never skip phases.** Execute Phase 0 through Phase 5 in exact order. Each phase has a GATE ENTRY and GATE EXIT — both must pass before proceeding.
- **Do not modify application code outside the worktree.** All code modifications must occur in `$WORKTREE`. Planning/design documents (`<PARENT_REPO>/docs/plans/`) and agent runtime results (`<PARENT_REPO>/.agents/results/`) are anchored to `<PARENT_REPO>` so they persist across worktree lifecycles.
- **Deterministic Child Framework Ownership (Guardrail 1)**: Phase 3 deterministically adopts `ultrawork` in Plan-Ingestion Mode. Never prompt the user to choose an implementation engine mid-flight. Ingest the approved Phase 2 plan JSON directly and execute `ultrawork`'s full IMPL → VERIFY → REFINE → SHIP lifecycle.
- **Fast-Fail Local CI Sanity Gate (Guardrail 2)**: Before committing changes or creating a pull request in Phase 4, the orchestrator must execute a local non-interactive CI sanity run (lint, typecheck, tests) in `$WORKTREE`.
- **Absolute Path Anchoring (Guardrail 3)**: Canonical absolute paths (`$PARENT_REPO`, `$WORKTREE`, `$RESULTS_DIR`) must be resolved in Phase 0 and passed to all subagents and child frameworks by reference to prevent ephemeral state loss upon worktree teardown.
- **Worktree Preservation Invariant (Guardrail 4)**: If any phase fails, encounters an error, or is blocked, the orchestrator is **STRICTLY FORBIDDEN** from executing `git worktree remove`. The worktree must remain intact on disk for reproduction, inspection, and manual debugging.
- **Provider-agnostic.** Detect GitHub (`gh`) vs GitLab (`glab`) from `origin` per `.agents/skills/_shared/runtime/providers.md`. Never hardcode one CLI.
- **Strictly follow ALL rules in the project's `AGENTS.md` and `TESTING.md`** (if they exist).
- **Phase ordering is inviolable.** Never reorder, skip, parallelize, or combine phases.
- **MUST use the question tool to ask the user anything.** Never use plain text output for user gates or clarification questions.
- **MUST use subagent tools for delegated subagents.** All Task subagent delegations (Phase 4 commit messages/PR descriptions, Phase 5 issue comment, and Phase 3 implementation/QA/debug agents) must be dispatched cleanly via subagent tools.
- **Subagent Dispatch Gate (HARD INVARIANT)**: Every Task subagent spawn MUST record its harness-returned `task_id` in `${RESULTS_DIR}/subagent-ledger-${sessionId}.json`. Before consuming any subagent's deliverable (Phase 4 Step 2, Phase 5), pass the **Subagent Dispatch Gate** (`.agents/skills/_shared/runtime/subagent-dispatch-gate.md`): non-empty `task_id`, `status == complete`, and a non-empty result file. On failure, dispatch the missing subagent — do NOT substitute inline work.
- **No Inline Substitution (HARD INVARIANT)**: A subagent's deliverable is defined as a file written by a spawned subagent whose `task_id` is recorded in the ledger. Orchestrator-inline output does not count as a subagent's deliverable, regardless of quality.
- **Rule loading (MANDATORY)**: instruct each spawned subagent to load before starting: `.agents/rules/grug-principles.md`, `.agents/rules/tool-compatibility.md`, and `.agents/skills/_shared/core/quality-principles.md`.
- **Subagents are cheap; use them aggressively.** Spawn focused implementation, review, and fix agents rather than doing everything inline.
- **Enforce Zero-Context Relay and File-First State I/O**: Pass context by file path reference; subagents write full deliverables to designated markdown files in `${RESULTS_DIR}` and return standardized 4-line chat summaries.

## Rules to Load

Before starting, load and follow:
- `.agents/rules/grug-principles.md` — universal engineering rules
- `.agents/rules/tool-compatibility.md` — cross-harness tool naming
- `.agents/skills/_shared/core/quality-principles.md` — quality principles
- `.agents/skills/_shared/core/context-loading.md` — resource loading strategy

---

## Phase Delegation Strategy

| Phase | Who Runs | Description |
|:---|:---|:---|
| **Phase 0: Init** | Orchestrator inline | Provider detection, auth verification, issue ingestion, path anchoring, worktree creation |
| **Phase 1: Brainstorm** | Orchestrator inline (invokes `brainstorm`) | Interactive design ideation, approach comparison, and design doc creation |
| **Phase 2: Plan** | Orchestrator inline (invokes `plan`) | Requirements breakdown, contract definition, task planning, and user confirmation |
| **Phase 3: Implement** | **Child Framework (`ultrawork` in Plan-Ingestion Mode)** | Full multi-agent lifecycle: IMPL (Step 5) → VERIFY (Steps 6–8) → REFINE (Steps 9–13) → SHIP (Steps 14–17) |
| **Phase 4: Forge Ship** | Task Subagent + Orchestrator inline | Fast-Fail Local CI Gate (inline), Conventional Commit & PR body (Subagent), Git push & Draft PR creation (inline), Worktree teardown (inline) |
| **Phase 5: Issue Comment** | **Task Subagent** | Plain-English non-technical summary generation and forge issue comment posting |

---

## The 4 Blind Review Guardrails

This workflow embeds 4 architectural guardrails to guarantee determinism, quality, and state persistence:

1. **Guardrail 1: Plan-Ingestion Mode (Double-Planning Prevention)**
   - In Phase 3, `issue-autopilot` passes the approved `${RESULTS_DIR}/plan-{sessionId}.json` from Phase 2 directly into `ultrawork`.
   - `ultrawork` ingests the pre-approved plan, skips its redundant internal Phase 1 PM re-planning (Steps 1–4) and duplicate human confirmation (`PLAN_GATE`), and immediately starts Phase 2 (IMPL: Step 5 parallel subagents).

2. **Guardrail 2: Fast-Fail Local CI Sanity Gate (Refinement Drift & Quality Barrier)**
   - In Phase 4 (Step 1), before committing or opening a PR, the orchestrator discovers and executes local test, lint, and typecheck commands in `$WORKTREE`.
   - Protects against syntax errors or regressions introduced during `ultrawork` Phase 4 (`REFINE`). If checks fail, a Debug Agent is spawned for root-cause remediation; if failing, PR creation is blocked.

3. **Guardrail 3: Absolute Path Anchoring (Worktree State Preservation)**
   - Resolves canonical absolute paths (`PARENT_REPO`, `WORKTREE`, `RESULTS_DIR`) during Phase 0.
   - All design documents (`docs/plans/designs/`), task plans (`docs/plans/work/`), and execution results (`.agents/results/`) are stored in `<PARENT_REPO>` so that worktree teardown never deletes audit trails or artifacts.

4. **Guardrail 4: Worktree Preservation Invariant (Safe Failure State)**
   - If any phase fails or is blocked, worktree removal is strictly forbidden. The worktree remains on disk for developer reproduction and manual intervention.

---

## Phase 0: Init

### GATE ENTRY
- User has a forge issue (GitHub or GitLab) to resolve.

### Procedure

1. Ask the user for the issue number or URL if not provided. **Use the question tool — not plain text.**
2. Detect provider from `git remote get-url origin` per `.agents/skills/_shared/runtime/providers.md`. Record `PROVIDER` (`github`|`gitlab`) and matching CLI (`gh`|`glab`).
3. Verify the provider CLI is authenticated with a functional request against the repo (`gh repo view` or `glab repo view`). If the request fails, report error and abort.
4. Fetch issue details using the Issues table in `.agents/skills/_shared/runtime/providers.md` (normalize title, body, labels, comments, assignees, state).
5. Present the issue to the user: title, body, labels, key comments.
6. Fetch latest changes to main and ensure local main is up to date:
   ```bash
   git fetch origin main
   git checkout main
   git pull origin main
   ```
7. Generate a branch name from the issue title: `feat-<kebab-title>-<number>` (or `fix-...`).
8. **Resolve and export canonical absolute paths (Guardrail 3)**:
   ```bash
   PARENT_REPO=$(git rev-parse --show-toplevel)
   WORKTREE=$(realpath ../<repo-name>-<number>)
   RESULTS_DIR="${PARENT_REPO}/.agents/results"
   ```
9. Create a git worktree from main:
   ```bash
   git worktree add "$WORKTREE" -b "$BRANCH" main
   ```
10. Record `$WORKTREE`, `$PARENT_REPO`, `$RESULTS_DIR`, `$BRANCH`, and `$PROVIDER`. Pass these canonical paths to all subsequent phases and subagents.

### GATE EXIT
- [ ] Provider detected and CLI authenticated
- [ ] Issue details captured and normalized
- [ ] Local main is up to date
- [ ] Canonical paths resolved (`$PARENT_REPO`, `$WORKTREE`, `$RESULTS_DIR`)
- [ ] Worktree created at `$WORKTREE` on branch `$BRANCH`
- [ ] `PROVIDER` and `CLI` recorded

**You CANNOT proceed to Phase 1 without satisfying ALL gate exit items.**

---

## Phase 1: Brainstorm

### GATE ENTRY
- [ ] Phase 0 gate exit conditions satisfied
- [ ] `$WORKTREE` is valid and accessible

### Procedure

1. `cd $WORKTREE` — enforce worktree isolation for codebase exploration.
2. Present the issue to the user as context.
3. Load the **brainstorm** skill (`.agents/skills/brainstorm/SKILL.md`) and follow it step by step. **You MUST execute every step in the brainstorm skill in order. Do NOT shortcut, combine, or skip any step.**
4. Explore user intent, constraints, and evaluate 2–3 alternative approaches with trade-off analysis and a recommended option.
5. Save the approved design doc to `${PARENT_REPO}/docs/plans/designs/<NNN>-<issue-title>.md`, where `${PARENT_REPO}` is the main repository checkout (NOT `$WORKTREE`) — planning docs must persist worktree removal.

### GATE EXIT
- [ ] Design document saved at `${PARENT_REPO}/docs/plans/designs/<NNN>-<issue-title>.md` (in parent repo)
- [ ] User explicitly approved the design (Step 5 blind review round completed, Step 6 saved)
- [ ] No application code files modified outside `$WORKTREE`
- [ ] Brainstorm skill completed in full (all steps)

**You CANNOT proceed to Phase 2 without satisfying ALL gate exit items.**

---

## Phase 2: Plan

### GATE ENTRY
- [ ] Phase 1 gate exit conditions satisfied
- [ ] Approved design document exists at `${PARENT_REPO}/docs/plans/designs/`

### Procedure

1. `cd $WORKTREE`
2. Load the **plan** skill (`.agents/skills/plan/SKILL.md`) and follow it step by step. **You MUST execute every step in the plan skill in order. Do NOT shortcut, combine, or skip any step.**
3. Analyze requirements, define API contracts, and decompose work into prioritized tasks with explicit acceptance criteria and sizing.
4. Save the machine-readable plan to `${RESULTS_DIR}/plan-${sessionId}.json`.
5. Save the human-readable tracker to `${PARENT_REPO}/docs/plans/work/<NNN>-<issue-title>.md` (for Medium/Complex plans).
6. **You MUST get explicit user confirmation (ask the user) before proceeding to Phase 3.**

### GATE EXIT
- [ ] Machine-readable plan saved at `${RESULTS_DIR}/plan-${sessionId}.json`
- [ ] Human-readable tracker saved at `${PARENT_REPO}/docs/plans/work/<NNN>-<issue-title>.md`
- [ ] User explicitly confirmed the plan when asked
- [ ] No application code files modified outside `$WORKTREE`

**You CANNOT proceed to Phase 3 without satisfying ALL gate exit items.**

---

## Phase 3: Implement (Deterministic `ultrawork` Adoption)

### GATE ENTRY
- [ ] Phase 2 gate exit conditions satisfied
- [ ] Machine-readable plan exists at `${RESULTS_DIR}/plan-${sessionId}.json`
- [ ] User explicitly confirmed the plan in Phase 2

### Procedure

1. `cd $WORKTREE`
2. **Enforce Child Framework Ownership**: Adopt the **ultrawork** skill (`.agents/skills/ultrawork/SKILL.md`).
3. **Execute in Plan-Ingestion Mode (Guardrail 1)**:
   - Pass prerequisite artifacts and canonical paths by reference:
     - `PLAN_FILE="${RESULTS_DIR}/plan-${sessionId}.json"`
     - `RESULTS_DIR="${RESULTS_DIR}"`
     - `PARENT_REPO="${PARENT_REPO}"`
     - `WORKTREE="${WORKTREE}"`
   - `ultrawork` ingests `PLAN_FILE` directly, **skipping redundant Phase 1 PM re-planning (Steps 1–4)** and skipping duplicate human confirmation (`PLAN_GATE`).
4. `ultrawork` executes its child lifecycle in `$WORKTREE`:
   - **Phase 2: IMPL (Step 5)**: Dispatches parallel Implementation Agents (`backend`, `frontend`, etc.) using File-First State I/O and Zero-Context Relay.
   - **Phase 3: VERIFY (Steps 6–8)**: Dispatches QA Agent for Alignment Review (Step 6), Security/Bug Review (Step 7), and Improvement/Regression Review (Step 8). Enforces root-cause-first remediation.
   - **Phase 4: REFINE (Steps 9–13)**: Dispatches Debug Agent for splitting large files (>500 lines), reusability review, side-effect analysis, and dead code cleanup.
   - **Phase 5: SHIP (Steps 14–17)**: Dispatches QA Agent for final code quality checks, UX flow verification, cascade impact review, and deployment readiness review.
5. All child output artifacts (`result-*.md`, `session-ultrawork.md`, `experiment-ledger.md`) are persisted directly to `${RESULTS_DIR}`.
6. Await `ultrawork` completion return (`Status: SUCCESS`).
7. After implementation completes, verify no application code files were modified outside `$WORKTREE`.

### GATE EXIT
- [ ] `ultrawork` child framework completed all lifecycle phases (IMPL → VERIFY → REFINE → SHIP)
- [ ] All tasks in the plan are marked DONE
- [ ] Zero CRITICAL or HIGH severity defects
- [ ] Execution deliverables documented in `${RESULTS_DIR}/result-*.md`
- [ ] `ultrawork` returned `Status: SUCCESS`
- [ ] No application code files modified outside `$WORKTREE`

**You CANNOT proceed to Phase 4 without satisfying ALL gate exit items.**

---

## Phase 4: Forge Ship

**GUARD: Do not enter this phase unless Phase 3 (`ultrawork`) completed with `Status: SUCCESS`.**

### GATE ENTRY
- [ ] Phase 3 gate exit conditions satisfied
- [ ] All `ultrawork` verification and refinement checks passed
- [ ] `$WORKTREE` contains verified uncommitted changes

### Procedure

#### Step 1: Fast-Fail Local CI Sanity Gate (Guardrail 2 — Orchestrator Inline)

1. `cd $WORKTREE`
2. Discover project CI commands from `.github/workflows/`, `.gitlab-ci.yml`, and repository `AGENTS.md` / `TESTING.md` per Local CI config discovery in `.agents/skills/_shared/runtime/providers.md`.
3. Run local non-interactive static analysis and test suite inside `$WORKTREE`:
   - Install dependencies if required.
   - Run lint, type-check, and format checks (e.g. `npm run lint`, `ruff check .`, `tsc --noEmit`, `mypy .`).
   - Run full test suite (e.g. `npm test`, `pytest`, `cargo test`, `go test ./...`).
4. **Binary Gate Evaluation**:
   - **Pass (Exit Code 0)**: Proceed to Step 2.
   - **Fail (Exit Code != 0)**:
     - Auto-fix simple mechanical issues (e.g. `ruff format`, `prettier --write`) if safe.
     - For test failures or type errors, dispatch a targeted Debug Agent via the subagent tool with the failure logs to apply a root-cause fix.
     - Re-run the CI Sanity Gate.
     - If CI fails after retry: **DO NOT COMMIT. DO NOT PUSH.** Preserve worktree intact (Guardrail 4), halt, and report failure output to the user.

#### Step 2: Generate Conventional Commit & PR Description (Task Subagent)

**CRITICAL: Delegate commit message and PR description generation to a Task subagent. Do not write them inline.**

1. `cd $WORKTREE`
2. Spawn an SCM Task subagent per [Subagent Prompts](.agents/skills/issue-autopilot/resources/subagent-prompts.md#1-scm-specialist-subagent-prompt):
   - `Role`: `"SCM Specialist Agent"`
   - `Prompt`: Instruct agent to inspect `WORKTREE` diffs, adhere to `.agents/skills/scm/SKILL.md` Conventional Commits (`Closes #<number>`), write `/tmp/commit-msg.txt` and `/tmp/pr-body.txt`, and write execution technical summary to `${RESULTS_DIR}/result-scm-ship-${sessionId}.md`.
3. Record the harness-returned `task_id` for this subagent in `${RESULTS_DIR}/subagent-ledger-${sessionId}.json` (role `scm-ship`, `result_file` `${RESULTS_DIR}/result-scm-ship-${sessionId}.md`).
4. Pass the **Subagent Dispatch Gate** (`.agents/skills/_shared/runtime/subagent-dispatch-gate.md`): confirm non-empty `task_id`, `status == complete`, and `${RESULTS_DIR}/result-scm-ship-${sessionId}.md` exists and is non-empty. Also confirm `/tmp/commit-msg.txt` and `/tmp/pr-body.txt` exist. On gate failure, re-dispatch the SCM subagent — do NOT write the commit message inline.

#### Step 3: Git Commit, Push, and Draft PR Creation (Orchestrator Inline)

4. Commit using the generated message:
   ```bash
   git add <specific files>    # NEVER git add -A / git add .
   git commit -F /tmp/commit-msg.txt
   ```
5. Push to the remote:
   ```bash
   git push -u origin "$BRANCH"
   ```
6. Create a draft PR using the Create draft PR row in `.agents/skills/_shared/runtime/providers.md` for `$PROVIDER`:
   - **GitHub**:
     ```bash
     gh pr create --draft --base main --title "<title>" --body-file /tmp/pr-body.txt
     ```
   - **GitLab**:
     ```bash
     glab mr create --draft --target-branch main --title "<title>" --description "$(cat /tmp/pr-body.txt)"
     ```
7. Report the PR/MR URL to the user in chat.

#### Step 4: Cleanup (Orchestrator Inline)

8. Return to parent repo main branch:
   ```bash
   cd "$PARENT_REPO"
   git checkout main
   ```
9. Remove the isolated worktree now that changes are pushed and PR is open:
   ```bash
   git worktree remove "$WORKTREE"
   ```

### GATE EXIT
- [ ] Local CI Sanity Gate passed with 0 errors
- [ ] SCM subagent dispatched with `task_id` recorded in the subagent ledger and result file non-empty (Subagent Dispatch Gate)
- [ ] Commit created adhering to Conventional Commits with `Closes #<number>`
- [ ] Branch pushed to remote origin
- [ ] Draft PR/MR created on forge and URL reported to user
- [ ] Returned to `main` branch in parent repository
- [ ] Worktree safely removed

**Do NOT proceed to Phase 5 if PR creation failed. Report the error and provide the manual create command from `.agents/skills/_shared/runtime/providers.md`.**

---

## Phase 5: Issue Comment — Task Subagent

**If the original issue number is unknown, skip this phase and report to the user.**

### GATE ENTRY
- [ ] Phase 4 gate exit conditions satisfied
- [ ] PR/MR created successfully
- [ ] Original issue number is known

### Procedure

**CRITICAL: Delegate this entire phase to a Task subagent. Do not write the comment inline.**

1. Spawn a Task subagent per [Subagent Prompts](.agents/skills/issue-autopilot/resources/subagent-prompts.md#2-issue-communicator-subagent-prompt):
   - `Role`: `"Issue Communicator Agent"`
   - `Prompt`: Instruct agent to read `.agents/skills/_shared/runtime/providers.md`, inspect merged diff from `$PARENT_REPO`, write non-technical plain-English summary (no code references or technical jargon) to `/tmp/issue-comment.txt`, post using provider CLI command, and write technical summary to `${RESULTS_DIR}/result-issue-comment-${sessionId}.md`.
2. Record the harness-returned `task_id` for this subagent in `${RESULTS_DIR}/subagent-ledger-${sessionId}.json` (role `issue-comment`, `result_file` `${RESULTS_DIR}/result-issue-comment-${sessionId}.md`).
3. Pass the **Subagent Dispatch Gate** (`.agents/skills/_shared/runtime/subagent-dispatch-gate.md`): confirm non-empty `task_id`, `status == complete`, and `${RESULTS_DIR}/result-issue-comment-${sessionId}.md` exists and is non-empty. On gate failure, re-dispatch — do NOT write the comment inline.
4. Present the posted comment text in chat to the user.

### GATE EXIT
- [ ] Issue Communicator subagent dispatched with `task_id` recorded in the subagent ledger and result file non-empty (Subagent Dispatch Gate)
- [ ] Plain-English comment posted to the forge issue
- [ ] User can see the comment content in chat
- [ ] Workflow complete

---

## Failure Handling

| Phase | Failure Mode | Automated Recovery Action | Fallback / Human Escalation |
|:---|:---|:---|:---|
| **0: Init** | Provider CLI not authenticated (`gh`/`glab`) | None (do not attempt auth modification). | Abort immediately. Instruct user to run `gh auth login` or `glab auth login`. |
| **0: Init** | Issue not found or invalid ID | None. | Report error to user and prompt for valid issue number. |
| **0: Init** | Worktree directory collision | Check if stale worktree exists via `git worktree list`. | If stale, prompt user to prune or use alternative suffix. |
| **1: Brainstorm** | User rejects proposed designs | Capture feedback and generate alternative approach. | If 3 consecutive rejections, cleanly abort and prune worktree. |
| **2: Plan** | User rejects plan | Modify task breakdown or sizing per feedback. | Loop back to Phase 1 if architectural changes needed. |
| **3: Implement** | Subagent build or test failure | `ultrawork` re-spawns Dev Agent with error logs (up to 3 retries). | Escalate to QA Review if unresolvable. |
| **3: Implement** | VERIFY gate failure (Regression / Safety) | `ultrawork` activates root-cause remediation prompt to Dev Agent. | If 2nd failure on same issue, trigger Exploration Loop. |
| **3: Implement** | Review loop cap (5 cycles) or cost cap hit | `ultrawork` halts, writes diagnostics to `session-ultrawork.md`. | **Guardrail 4: Preserve worktree intact.** Report roadblock in chat. |
| **4: Forge Ship** | Fast-Fail Local CI Gate fails | Spawn targeted Debug Agent to fix lint/type/test failures. | If fix fails, **DO NOT COMMIT**. Leave worktree intact and alert user. |
| **4: Forge Ship** | Remote git push rejected | Fetch and rebase against latest `origin/main` in worktree. | If conflicts arise, spawn merge subagent or alert user. |
| **4: Forge Ship** | Draft PR creation fails | Verify branch pushed; retry with explicit `-R OWNER/REPO`. | Branch is pushed; provide manual CLI create command to user. |
| **4: Forge Ship** | SCM Subagent fails to generate message | Fallback: construct Conventional Commit message inline. | Proceed with commit and PR creation. |
| **5: Comment** | Original issue number unknown | Skip phase automatically. | Notify user in chat that PR is created but comment was skipped. |
| **5: Comment** | Issue comment post fails | Retry once after 5s backoff; save comment to local file. | Output comment in chat for manual copy-paste. PR remains intact. |

> **Universal Worktree Preservation Invariant (Guardrail 4)**:
> If any phase after Phase 0 terminates in `FAILED` or `BLOCKED` state, the orchestrator is **STRICTLY FORBIDDEN** from executing `git worktree remove`. The worktree must remain intact on disk to allow developer reproduction, inspection, and manual debugging. Only successful completion of Phase 4 worktree teardown or explicit user instruction authorizes worktree removal.

---

## Phase Transition Map

```
Phase 0: Init [inline]
       │
       ▼
Phase 1: Brainstorm [brainstorm skill]
       │
       ▼
Phase 2: Plan [plan skill + ask the user]
       │
       ▼
Phase 3: Implement [ultrawork in Plan-Ingestion Mode]
       │  (IMPL: Step 5 → VERIFY: Steps 6-8 → REFINE: Steps 9-13 → SHIP: Steps 14-17)
       │
       ▼
Phase 4: Forge Ship
       ├── Step 1: Fast-Fail Local CI Sanity Gate [inline]
       │     │
       │     ├─ FAIL ──► [Debug Subagent Fix] ──► Re-test (or HALT & Preserve Worktree)
       │     ▼ PASS
       ├── Step 2: Conventional Commit & PR Body [TASK Subagent]
       ├── Step 3: Git Commit, Push, Draft PR Create [inline]
       └── Step 4: Worktree Teardown [inline]
             │
             ▼
Phase 5: Issue Comment [TASK Subagent]
       │
       ▼
Workflow Complete
```

- **[inline]**: Executed directly by the orchestrator.
- **[TASK Subagent]**: Delegated to a background subagent via the subagent tool.
- **[ultrawork in Plan-Ingestion Mode]**: Ingests Phase 2 plan JSON, executes full child multi-agent lifecycle, and reports terminal status.

---

## References

- Tool compatibility (cross-harness tool names): `.agents/rules/tool-compatibility.md`
- Subagent Prompts: `.agents/skills/issue-autopilot/resources/subagent-prompts.md`
- Operational Runbooks: `.agents/skills/issue-autopilot/resources/operational-runbooks.md`
- Provider CLI map: `.agents/skills/_shared/runtime/providers.md`
- Implementation Framework: `.agents/skills/ultrawork/SKILL.md`
- Brainstorming Skill: `.agents/skills/brainstorm/SKILL.md`
- Planning Skill: `.agents/skills/plan/SKILL.md`
- SCM & Conventional Commits: `.agents/skills/scm/SKILL.md`
