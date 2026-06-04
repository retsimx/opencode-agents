---
description: Ultrawork - high-quality 5-phase development workflow with 11 review steps out of 17
---

# MANDATORY RULES: VIOLATION IS FORBIDDEN

- **NEVER skip steps.** Execute from Step 0 in order. Explicitly report completion of each step to the user before proceeding to the next.
- Use OpenCode's built-in tools for all operations:
  - `read`, `write`, `edit`, `grep`, `glob`, `bash` for code exploration and file operations
  - Use `.agents/results/` for all coordination and progress files
  - Do NOT rely on MCP-specific tools or memory providers
- **Read the coordination skill BEFORE starting.** Read `.agents/skills/coordination/SKILL.md` and follow its Core Rules.
- **Follow the context-loading guide.** Read `.agents/skills/_shared/core/context-loading.md` and load only task-relevant resources.

---


---

## Phase 0: Initialization (DO NOT SKIP)

1. Read `.agents/skills/coordination/SKILL.md` and confirm Core Rules.
2. Read `.agents/skills/_shared/core/context-loading.md` for resource loading strategy.
3. Read `.agents/skills/_shared/runtime/coordination-protocol.md` for the coordination protocol.
4. Read `.agents/workflows/ultrawork/resources/multi-review-protocol.md` (11 review guides)
5. Read `.agents/skills/_shared/core/quality-principles.md` (4 principles)
6. Read `.agents/workflows/ultrawork/resources/phase-gates.md` (gate definitions)
7. Record session start:
   - Write `.agents/results/session-ultrawork.md`
   - Include: session start time, user request summary, workflow version (ultrawork)

---

## Phase 1: PLAN (Steps 1-4)

### Step 1: Create Plan & Review
// turbo
Activate PM Agent to execute Steps 1-4:

1. Analyze requirements.
2. Define API contracts.
3. Create a prioritized task breakdown.
4. Execute Plan Review - Completeness (Step 2).
5. Execute Meta Review (Step 3).
6. Execute Over-Engineering Review (Step 4).
7. Save plan to `.agents/results/plan-{sessionId}.json`.
8. Create `task-board.md` in `.agents/results/` for dashboard compatibility.
9. Write plan completion to `.agents/results/session-ultrawork.md`.

### Step 2: Plan Review (Completeness)
- **Executed by PM Agent**: Ensure requirements are fully mapped.

### Step 3: Review Verification (Meta Review)
- **Executed by PM Agent**: Self-verify if the review was sufficient.

### Step 4: Over-Engineering Review (Simplicity)
- **Executed by PM Agent**: Check for unnecessary complexity (MVP focus).

### PLAN_GATE
- [ ] Plan documented
- [ ] Assumptions listed
- [ ] Alternatives considered
- [ ] Over-engineering review done
- [ ] **User confirmation**

**On gate pass**: Update `.agents/results/session-ultrawork.md` phase completion

**Gate failure → Return to Step 1**

---

## Phase 2: IMPL (Step 5)

### Step 5: Implementation
// turbo
Spawn Implementation Agents (Backend/Frontend/Mobile) in parallel.

### Dispatch via OpenCode Task tool

Spawn implementation agents in parallel using the OpenCode Task tool:
- Use `subagent_type="general"` for all implementation agents
- Include execution protocol, task description, and context-loading rules in the prompt
- Spawn all same-priority agents in a single message for parallel execution

---

### Step 5.1: Monitor & Wait for Completion

**Wait for all implementation agents to complete before proceeding.**

1. Read `.agents/results/progress-{agent}[-{sessionId}].md` files
2. Use `grep` and `glob` to verify implementation alignment
3. Check for `result-{agent}[-{sessionId}].md` files to confirm completion
4. Update `.agents/results/session-ultrawork.md` with monitoring results

**Continue polling until all agents report completion or failure.**

### Step 5.2: Measure Baseline Quality Score (Conditional)

If automated measurement is available (tests, lint exist):

1. Load `quality-score.md` (conditional, per `context-loading.md`)
2. Run tests, lint, type-check via Bash to measure baseline
3. Create Experiment Ledger: write `.agents/results/experiment-ledger.md` with initial baseline row
4. Record composite score as the IMPL baseline

If no measurement tools: skip; gates fall back to binary checklist.

### IMPL_GATE
- [ ] Build succeeds
- [ ] Tests pass
- [ ] Only planned files modified
- [ ] (If measured) Baseline Quality Score recorded in Experiment Ledger

**On gate pass**: Update `.agents/results/session-ultrawork.md` phase completion

**Gate failure → Return to Step 5, re-spawn failed agents, and repeat monitoring until GATE passes.**

---

## Phase 3: VERIFY (Steps 6-8)

### Step 6-8: QA Verification
// turbo
Spawn QA Agent to execute Steps 6-8.

Spawn QA Agent via OpenCode Task tool (subagent_type="general").

---

### Monitor QA Agent Progress

**Wait for QA Agent to complete verification before proceeding.**

1. Read `.agents/results/progress-qa-agent[-{sessionId}].md`
2. Check for `.agents/results/result-qa-agent[-{sessionId}].md` to confirm completion
3. Update `.agents/results/session-ultrawork.md` with QA results

**Continue polling until QA Agent reports completion.**

### Step 6: Alignment Review
- **Executed by QA Agent**: Compare implementation vs plan.

### Step 7: Security/Bug Review (Safety)
- **Executed by QA Agent**: Check for vulnerabilities (Safety).

### Step 8: Improvement Review (Regression Prevention)
- **Executed by QA Agent**: Run regression tests.

### Step 8.1: Measure Post-VERIFY Quality Score (Conditional)

If baseline was measured at Step 5.2:
1. Measure Quality Score incorporating QA findings
2. Calculate delta from IMPL baseline
3. Record as experiment in `.agents/results/experiment-ledger.md`

### VERIFY_GATE
- [ ] Implementation = Requirements
- [ ] CRITICAL count: 0
- [ ] HIGH count: 0
- [ ] No regressions
- [ ] (If measured) Quality Score >= 75 (Grade B)

**On gate pass**: Update `.agents/results/session-ultrawork.md` phase completion

**Gate failure (1st time)** → Before re-spawning for the next VERIFY cycle, check the session cost cap:

> **Review Loop termination conditions** (OR, whichever fires first wins):
> 1. Gate failure count has reached the configured maximum iterations (default: 5 total VERIFY + REFINE cycles). Do not start another cycle.
> 2. Session cost cap exceeded: call `checkCap(sessionId, loadQuotaCap())` from `cli/io/session-cost.ts`. If `exceeded === true`, print `formatPromptMessage(result)` to the user and stop the loop immediately. Save all current step results before stopping, then report to the user that the loop was terminated early due to quota.
>
> If neither condition is met, return to Step 5 and continue.

**Root-cause-first fix mandate:** when re-spawning implementation agents to address QA findings, the fix prompt MUST require root-cause remediation. Forbid tactical patches (try/catch swallowing the error, validation bypass, hardcoded values, feature flags hiding the bug, silencing the failing test) unless the agent explicitly justifies why a structural fix is out of scope (upstream library bug, deprecated path, hotfix window).

**Gate failure (2nd time on same issue, and termination conditions not yet met)** → Activate **Exploration Loop**:
1. Load `exploration-loop.md` (conditional, per `context-loading.md`)
2. Generate 2-3 alternative hypotheses using Exploration Decision template (`reasoning-templates.md` #6)
3. Experiment each approach sequentially (git stash per attempt)
4. Measure Quality Score for each
5. Select the highest-scoring approach
6. Record all experiments in Experiment Ledger
7. Resume VERIFY with winning approach

---

## Phase 4: REFINE (Steps 9-13)

### Step 9-13: Deep Refinement
// turbo
Spawn Debug Agent (or Senior Dev Agent) to execute Steps 9-13.

Spawn Debug Agent via OpenCode Task tool (subagent_type="general").

---

### Monitor Debug Agent Progress

**Wait for Debug Agent to complete refinement before proceeding.**

1. Read `.agents/results/progress-debug-agent[-{sessionId}].md`
2. Check for `.agents/results/result-debug-agent[-{sessionId}].md` to confirm completion
3. Update `.agents/results/session-ultrawork.md` with refinement results

**Continue polling until Debug Agent reports completion.**

### Step 9: Split Large Files/Functions
- **Executed by Debug Agent**: Files > 500 lines, Functions > 50 lines.

### Step 10: Integration/Reuse Review (Reusability)
- **Executed by Debug Agent**: Check for duplicate logic.

### Step 11: Side Effect Review (Cascade Impact)
- **Executed by Debug Agent**: Analyze impact scope.

### Step 12: Full Change Review (Consistency)
- **Executed by Debug Agent**: Review naming and style.

### Step 13: Clean Up Unused Code
- **Executed by Debug Agent**: Remove newly created dead code.

### Step 13.1: Measure Post-REFINE Quality Score (Conditional)

If baseline was measured at Step 5.2:
1. Measure Quality Score after refinement
2. Calculate delta from Post-VERIFY score
3. **If delta < -5**: Apply Discard rule. Revert refinement changes, record in Experiment Ledger.
4. Record kept experiments in Experiment Ledger

### REFINE_GATE
- [ ] No large files/functions
- [ ] Integration opportunities captured
- [ ] Side effects verified
- [ ] Code cleaned
- [ ] (If measured) Quality Score >= Post-VERIFY score (no regression from refinement)

**On gate pass**: Update `.agents/results/session-ultrawork.md` phase completion

**Gate failure → Before re-spawning the Debug Agent, apply the same termination check:**

> **Review Loop termination conditions** (OR, whichever fires first wins):
> 1. Total REFINE failure count has reached the configured maximum iterations (default: 5 cycles across all phases). Do not start another cycle.
> 2. Session cost cap exceeded: call `checkCap(sessionId, loadQuotaCap())` from `cli/io/session-cost.ts`. If `exceeded === true`, print `formatPromptMessage(result)` to the user and stop. Save current step results before stopping, then report early termination due to quota.
>
> If neither condition is met, re-spawn the Debug Agent with specific issues and repeat until GATE passes.

**Skip conditions**: Simple tasks < 50 lines

---

## Phase 5: SHIP (Steps 14-17)

### Step 14-17: Final QA & Deployment Readiness
// turbo
Spawn QA Agent to execute Steps 14-17.

Spawn QA Agent via OpenCode Task tool (subagent_type="general").

---

### Monitor Final QA Progress

**Wait for QA Agent to complete final review before proceeding.**

1. Read `.agents/results/progress-qa-agent[-{sessionId}].md`
2. Check for `.agents/results/result-qa-agent[-{sessionId}].md` to confirm completion
3. Update `.agents/results/session-ultrawork.md` with final QA results

**Continue polling until QA Agent reports completion.**

### Step 14: Code Quality Review
- **Executed by QA Agent**: Lint, Types, Coverage.

### Step 15: UX Flow Verification
- **Executed by QA Agent**: User journey check.

### Step 16: Related Issues Review (Cascade Impact 2nd)
- **Executed by QA Agent**: Final impact check.

### Step 17: Deployment Readiness Review (Final)
- **Executed by QA Agent**: Secrets, Migrations, checklist.

### Step 17.1: Final Quality Score & Session Summary (Conditional)

If Quality Score was measured during this session:
1. Measure final Quality Score
2. Generate Experiment Ledger summary (total experiments, keep rate, net delta)
3. Auto-generate lessons from discarded experiments (delta <= -5) into `lessons-learned.md`
4. Append Quality Score Progression and Experiment Summary to session metrics

**Always** (regardless of Quality Score availability):
5. Record Evaluator Accuracy events for this session:
   - Review all QA findings: any disputed by impl agents? → `false_positive`
   - Review runtime verification results: any stubs caught that static review missed? → `missed_stub`
   - Review impl agent self-check results: any bugs caught by QA that self-check missed? → `good_catch`
6. Append EA events to `session-metrics.md`
7. If rolling 3-session EA >= 30: Flag in final report
   → "QA tuning suggested. Run `retro` to review."

### SHIP_GATE
- [ ] Quality checks pass
- [ ] UX verified
- [ ] Related issues resolved
- [ ] Deployment checklist complete
- [ ] (If measured) Final Quality Score >= 75 (Grade B) with non-negative delta from baseline
- [ ] (If measured) Experiment Ledger summary recorded
- [ ] **User final approval**

**On gate pass**: Write final results in `session-ultrawork.md`

**Gate failure → Address issues, re-run affected steps, and repeat until GATE passes.**

---

## Step 18: Optional Doc Verify Hook

To run docs verification after changes, load the `docs` skill and use `docs-verify`.
This hook is opt-in; skip by default.

---

## Review Steps Summary

| Phase  | Steps | Agent       | Execution | Perspective                       |
| ------ | ----- | ----------- | --------- | --------------------------------- |
| PLAN   | 1-4   | PM Agent    | Inline    | Completeness, Meta, Simplicity    |
| IMPL   | 5     | Dev Agents  | Spawn     | Implementation                    |
| VERIFY | 6-8   | QA Agent    | Spawn     | Alignment, Safety, Regression     |
| REFINE | 9-13  | Debug Agent | Spawn     | Reusability, Cascade, Consistency |
| SHIP   | 14-17 | QA Agent    | Spawn     | Quality, UX, Cascade 2nd, Deploy  |

**Total 11 review steps + conditional Quality Score checkpoints → High quality guaranteed**

---

## Autoresearch-Inspired Enhancements

This workflow conditionally incorporates patterns from autoresearch:

| Pattern | When Active | Reference |
|---------|-------------|-----------|
| **Continuous metrics** | When measurement tools available | `quality-score.md` (loaded at VERIFY/SHIP) |
| **Keep/Discard** | When quality score is measured | `quality-score.md` delta rules |
| **Experiment logging** | When baseline is established | `experiment-ledger.md` (via memory protocol) |
| **Hypothesis exploration** | On repeated gate failures | `exploration-loop.md` (loaded on trigger) |
| **Auto-learning** | At session end, if experiments exist | `lessons-learned.md` auto-generation |

All protocols are loaded **conditionally** per `context-loading.md`, not at Phase 0.
