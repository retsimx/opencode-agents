---
name: orchestrate
description: Automated parallel agent execution that spawns subagents via OpenCode `task` tool and coordinates through file-based progress tracking. Use for orchestration, parallel execution, and automated multi-agent workflows.
---

# MANDATORY RULES: VIOLATION IS FORBIDDEN

- **NEVER skip steps.** Execute from Step 0 in order. Explicitly report completion of each step before proceeding.
- **Use Task subagents for isolated work** — delegate distinct subtasks to subagents rather than doing everything inline. Subagents are cheap; they prevent context dilution and scope creep.
- **Use the `question` tool when uncertain** — never make assumptions. Guessing leads to wasted work. Ask a quick question instead.
- Use OpenCode's built-in tools for all operations:
  - `read`, `write`, `edit`, `grep`, `glob`, `bash` for code exploration and file operations
  - Use `.agents/results/` for all coordination and progress files
  - Do NOT rely on MCP-specific tools or memory providers
- **Read required documents BEFORE starting.**

---

## Scheduling

### Goal
Automatically orchestrate multi-agent execution with task decomposition, parallel agent spawning, file-based progress tracking, verification, QA cross-review, retry, and result collection.

### Intent signature
- User asks to orchestrate, run in parallel, automate multi-agent execution, or coordinate full-stack work end to end.
- Task requires multiple specialist agents and a persistent review/remediation loop.

### When to use
- Complex feature requires multiple specialized agents working in parallel
- User wants automated execution without manually spawning agents
- Full-stack implementation spanning backend, frontend, mobile, and QA
- User says "run it automatically", "run in parallel", or similar automation requests

### When NOT to use
- Simple single-domain task -> use the specific agent directly
- User wants step-by-step manual control -> use coordination
- Quick bug fixes or minor changes

### Expected inputs
- Complex feature or workflow request
- Agent types, task constraints, and workspace/session needs
- Acceptance criteria and verification expectations

### Expected outputs
- Session state, task board, progress files, result files, and final summary
- Specialist agent outputs after mechanical checks, automated verify, and QA cross-review
- Review history and retry/remediation status when loops fail

### Dependencies
- OpenCode `task` tool (built-in)
- Subagent prompt template, scripts, task templates, verify script, and session metrics

### Control-flow features
- Branches by priority tiers, agent completion/failure, verification status, QA verdict, retry limits, and clarification debt
- Spawns agents via the `task` tool and reads/writes coordination files
- Blocks termination until persistent workflows complete

---

## Procedural Execution Steps

### Step 0: Preparation (DO NOT SKIP)

1. Load the coordination skill and confirm Core Rules.
2. Load the context-loading skill for resource loading strategy.
3. Load the coordination-protocol skill for the coordination protocol.

---

### Step 1: Load or Create Plan

Look for a plan file:

1. Check `.agents/results/plan-{sessionId}.json` (current session's plan).
2. If not found: find the most recent `.agents/results/plan-*.json` file.
3. If none exist: ask the user to run the plan skill first, or ask them to describe the tasks to execute.
- **Do NOT proceed without a plan.**

---

### Step 2: Initialize Session

// turbo

1. Generate session ID (format: `session-YYYYMMDD-HHMMSS`).
2. Write `orchestrator-session.md` and `task-board.md` to `.agents/results/`.
3. Set session status to RUNNING.

---

### Step 3: Spawn Agents by Priority Tier

// turbo
For each priority tier (P0 first, then P1, etc.):

- Each agent gets: task description, API contracts, relevant context from `_shared/core/context-loading.md`.
- Use `edit` to update `.agents/results/task-board.md` with agent status.

#### Dispatch via OpenCode `task` tool

Spawn agents using the OpenCode `task` tool. All same-priority tasks should be spawned in parallel (multiple `task` tool calls in one message).

| Domain | subagent_type |
|:-------|:--------------|
| backend | general |
| frontend | general |
| mobile | general |
| db | general |
| qa | general |
| debug | general |
| pm | general |
| architecture | general |
| tf-infra | general |
| docs | explore |

Each `task` tool invocation receives:
- `description`: short label
- `subagent_type`: "general" (or "explore" for docs/analysis)
- `prompt`: task description + execution protocol + relevant context + API contracts

---

### Step 4: Monitor Progress

Use `read` to check `.agents/results/progress-{agent}.md` for logic updates.

- Use `edit` to update `.agents/results/task-board.md` with turn counts and status changes.
- Watch for: completion, failures, crashes.

#### Context Anxiety Check (per polling cycle)

At each poll, evaluate for every in-progress agent:

1. **Turn budget ratio**: `turns_used / expected_turns` from difficulty guide
2. **Progress ratio**: `completed_criteria / total_criteria` from task-board

| Turn Budget | Progress | Action |
|-------------|----------|--------|
| < 80% | any | Continue monitoring |
| >= 80% | >= 50% | Continue (agent is on track to finish) |
| >= 80% | < 50% | **Context Reset**: Checkpoint + re-spawn (see `_shared/core/context-budget.md`) |
| 100% (max turns) | < 100% | **Context Reset**: Force checkpoint + re-spawn with remaining items |

Record reset events in `task-board.md`:
```
| Agent | Status | Note |
| backend | reset-1 | Turn budget 80%, progress 40%, checkpoint saved |
```

---

### Step 5: Verify Completed Agents

// turbo
For each completed agent, run automated verification:

1. Run project lint/typecheck: `poetry run python manage.py lint ...` or the project's equivalent.
2. Run project test suite: `poetry run python manage.py test ...` or the project's equivalent.
3. If verification commands are not found, check `README.md` or `AGENTS.md` for the correct commands.

- PASS (exit 0): accept result. If Quality Score is active, measure and record in Experiment Ledger.
- FAIL (exit 1): Before re-spawning, apply the Review Loop termination check:

  > **Review Loop termination conditions** (OR, whichever fires first wins):
  > 1. Retry count for this agent has reached the configured maximum (default: 2 retries). Do not start another retry cycle.
  > 2. Session cost cap exceeded: check the configured budget. If exceeded, save the current agent's partial results before stopping, then report early termination due to quota. Do not spawn the next retry or any remaining agents in the tier.
  >
  > If neither condition is met, re-spawn the agent with error context and increment the retry counter.

- FAIL (after 2 retries, and cost cap not yet exceeded): Activate **Exploration Loop** (load `exploration-loop.md` per `context-loading.md`):
  1. Generate 2-3 alternative hypotheses for the failing task
  2. Spawn the **same agent type** with different hypothesis prompts (parallel, separate workspaces)
  3. Score each result with Quality Score (if available)
  4. Keep the highest-scoring approach, discard others
  5. Record all experiments in Experiment Ledger

---

### Step 6: Collect Results

// turbo
After all agents complete, use `read` to read all `.agents/results/result-{agent}-{sessionId}.md` files.
Compile summary: completed tasks, failed tasks, files changed, remaining issues.

---

### Step 7: Final Report

Present session summary to the user.

- If any tasks failed after retries, list them with error details.
- Suggest next steps: manual fix, re-run specific agents, or run the review skill for QA.
- Use memory write tool to record final results.
- If Quality Score was measured during this session:
  - Generate Experiment Ledger summary (total experiments, keep rate, net delta)
  - Auto-generate lessons from discarded experiments (delta <= -5) into `lessons-learned.md`
  - Include agent effectiveness ranking in the report

---

## Agent-to-Agent Review Loop

After each agent completes Step 5 verification, enter an iterative review loop, not a single-pass check.

### Loop Flow

```
Agent completes work
    ↓
[1] Mechanical Self-Check: lint, type-check, tests, diff scope
    ↓
[2] Verify: Run the agent's verification commands (build, test, lint)
    ↓ FAIL → Agent receives feedback, fixes, back to [1]
    ↓ PASS
[3] Cross-Review: QA agent reviews the changes
    ↓ FAIL → Agent receives review feedback, fixes, back to [1]
    ↓ PASS
Accept result
```

### Step Details

**[1] Mechanical Self-Check** (formerly "Self-Review"):
Before requesting external review, the implementation agent must:
- Run lint, type-check, and tests in the workspace
- Verify only planned files were modified (diff scope check)
- Fix any mechanical failures (compile errors, test failures)

**Quality judgment is NOT performed in this step.**
Design quality, architecture alignment, and acceptance criteria satisfaction
are evaluated exclusively in [3] Cross-Review by the QA agent.
Reason: Self-evaluation bias causes agents to consistently overrate their own output.

**[2] Automated Verify**:
- Run the agent's project verification commands (build, test, lint)
- **PASS (exit 0)**: Proceed to cross-review
- **FAIL (exit 1)**: Feed verify output back to the agent as correction context

**[3] Cross-Review**: Spawn QA agent to review the changes:
- QA agent reads the diff, runs checks, evaluates against acceptance criteria
- If `docs/CODE-REVIEW.md` exists, QA agent uses it as the review checklist
- QA agent outputs: PASS (with optional nits) or FAIL (with specific issues)
- On FAIL: issues are fed back to the implementation agent for fixing

### Loop Limits

| Counter | Max | On Exceeded |
|---------|-----|-------------|
| Self-check + fix cycles | 3 | Escalate to cross-review regardless |
| Cross-review rejections | 2 | Report to user with review history |
| Total loop iterations | 5 | Force-complete with quality warning |

### Review Feedback Format

When feeding review results back to the implementation agent:
```
## Review Feedback (iteration {n}/{max})
**Reviewer**: {self / verify / qa-agent}
**Verdict**: FAIL
**Issues**:
1. {specific issue with file and line reference}
2. {specific issue}
**Fix instruction**: {what to change}
```

This replaces single-pass verification. Most "nitpicking" should happen agent-to-agent.
Human review is reserved for final approval, not catching lint errors.

### Retry Logic (after review loop exhaustion)
- 1st retry: Re-spawn agent with full review history as context
- 2nd retry: Re-spawn with "Try a different approach" + review history
- Final failure: Report to user with complete review trail, ask whether to continue or abort

---

## Clarification Debt (CD) Monitoring

Track user corrections during session execution. See `../_shared/core/session-metrics.md` for full protocol.

### Event Classification
When user sends feedback during session:
- **clarify** (+10): User answering agent's question
- **correct** (+25): User correcting agent's misunderstanding
- **redo** (+40): User rejecting work, requesting restart

### Threshold Actions
| CD Score | Action |
|----------|--------|
| CD >= 50 | **RCA Required**: QA agent must add entry to `lessons-learned.md` |
| CD >= 80 | **Session Pause**: Request user to re-specify requirements |
| `redo` >= 2 | **Scope Lock**: Request explicit allowlist confirmation before continuing |

### Recording
After each user correction event, append event to `session-metrics.md` Events table.

At session end, if CD >= 50:
1. Include CD summary in final report
2. Trigger QA agent RCA generation
3. Update `lessons-learned.md` with prevention measures

---

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| MAX_PARALLEL | 3 | Max concurrent subagents |
| MAX_RETRIES | 2 | Retry attempts per failed task |
| POLL_INTERVAL | 30s | Status check interval |
| MAX_TURNS (impl) | 20 | Turn limit for backend/frontend/mobile |
| MAX_TURNS (review) | 15 | Turn limit for qa/debug |
| MAX_TURNS (plan) | 10 | Turn limit for pm |

---

## Coordination Files

All coordination is file-based in `.agents/results/`. See `resources/coordination-schema.md` for file formats.

### File Ownership

| File | Owner | Others |
|------|-------|--------|
| `orchestrator-session.md` | orchestrate | read-only |
| `task-board.md` | orchestrate | read-only |
| `progress-{agent}[-{sessionId}].md` | that agent | orchestrate reads |
| `result-{agent}[-{sessionId}].md` | that agent | orchestrate reads |

---

## Guardrails

1. Use the OpenCode `task` tool with subagent_type="general" for all agent spawning.
2. Never exceed the configured parallelism or retry limits.
3. Keep session state, task-board state, progress files, and result files aligned throughout the run.
4. Never reorder, skip, or combine the procedural Steps 0-7.

---

## References
- Prompt template: `resources/subagent-prompt-template.md`
- Coordination schema: `resources/coordination-schema.md`
- Config: `config/cli-config.yaml`
- Task templates: `templates/`
- Skill-to-agent mapping: `../_shared/core/skill-routing.md`
- Session metrics: `../_shared/core/session-metrics.md`
- API contracts: `../_shared/core/api-contracts/`
- Context loading: `../_shared/core/context-loading.md`
- Difficulty guide: `../_shared/core/difficulty-guide.md`
- Reasoning templates: `../_shared/core/reasoning-templates.md`
- Clarification protocol: `../_shared/core/clarification-protocol.md`
- Context budget: `../_shared/core/context-budget.md`
- Lessons learned: `../_shared/core/lessons-learned.md`
