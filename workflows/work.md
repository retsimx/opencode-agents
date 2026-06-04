---
description: Coordinate multiple agents for a complex multi-domain project using PM planning, parallel agent spawning, and QA review
---

# MANDATORY RULES: VIOLATION IS FORBIDDEN

- **NEVER skip steps.** Execute from Step 0 in order. Explicitly report completion of each step to the user before proceeding to the next.
- **Use Task subagents for isolated work** — delegate distinct subtasks to subagents rather than doing everything inline. Subagents are cheap; they prevent context dilution and scope creep.
- **Use the `question` tool when uncertain** — never make assumptions. Guessing leads to wasted work. Ask a quick question instead.
- Use OpenCode's built-in tools for all operations:
  - `read`, `write`, `edit`, `grep`, `glob`, `bash` for code exploration and file operations
  - Use `.agents/results/` for all coordination and progress files
  - Do NOT rely on MCP-specific tools or memory providers
- **Read the coordination skill BEFORE starting.** Read `.agents/skills/coordination/SKILL.md` and follow its Core Rules.
- **Follow the context-loading guide.** Read `.agents/skills/_shared/core/context-loading.md` and load only task-relevant resources.

---

## Step 0: Preparation (DO NOT SKIP)

1. Read `.agents/skills/coordination/SKILL.md` and confirm Core Rules.
2. Read `.agents/skills/_shared/core/context-loading.md` for resource loading strategy.
3. Read `.agents/skills/_shared/runtime/coordination-protocol.md` for the coordination protocol.
4. Record session start:
   - Write `.agents/results/session-work.md`
   - Include: session start time, user request summary.

---

## Step 1: Analyze Requirements

Analyze the user's request and identify involved domains (frontend, backend, mobile, QA).

- Single domain: suggest using the specific agent directly.
- Multiple domains: proceed to Step 2.
- Use `grep` and `glob` to understand the existing codebase structure relevant to the request.
- Report analysis results to the user.

---

## Step 2: Run PM Agent for Task Decomposition

// turbo
Activate PM Agent to:

1. Analyze requirements.
2. Define API contracts.
3. Create a prioritized task breakdown.
4. Save plan to `.agents/results/plan-{sessionId}.json`.
5. Record plan completion in `.agents/results/session-work.md`.

---

## Step 3: Review Plan with User

Present the PM Agent's task breakdown to the user:

- Priorities (P0, P1, P2)
- Agent assignments
- Dependencies
- **You MUST get user confirmation before proceeding to Step 4.** Do NOT proceed without confirmation.

---

## Step 4: Spawn Agents by Priority Tier

// turbo
Spawn agents for each task by priority tier (P0 first, then P1, etc.).
Spawn all same-priority tasks in parallel. Assign separate workspaces to avoid file conflicts.

### Dispatch via OpenCode Task tool

Spawn agents using the OpenCode Task tool:
- Use `subagent_type="general"` for implementation agents
- Use `subagent_type="explore"` for research/analysis agents
- Spawn all same-priority tasks in parallel (multiple Task tool calls in one message)
- Assign clear task descriptions in the prompt
- Include execution protocol instructions in the prompt

---

## Step 5: Monitor Agent Progress

- Use `read` to check `.agents/results/progress-{agent}.md` files
- Use `grep` and `glob` to verify API contract alignment between agents
- Record monitoring results in `.agents/results/session-work.md`

---

## Step 6: Run QA Agent Review

After all implementation agents complete, use the OpenCode Task tool to spawn a QA agent to review all deliverables:

- Security (OWASP Top 10)
- Performance
- Accessibility (WCAG 2.1 AA)
- Code quality

---

## Step 6.1: Measure Quality Score (Conditional)

If automated measurement is available:
1. Load `quality-score.md` (conditional, per `context-loading.md`)
2. Measure Quality Score based on QA findings
3. Record as baseline in Experiment Ledger via memory tools

---

## Step 7: Address Issues and Iterate

If QA finds CRITICAL or HIGH issues:

1. Re-spawn the responsible agent with QA findings. **The fix prompt MUST instruct root-cause remediation, not symptom suppression.** Forbid tactical patches (try/catch swallowing, validation bypass, hardcoded values, feature flags hiding the bug, silencing the failing test) unless the agent can explicitly justify why a structural fix is out of scope for this iteration (e.g., upstream library bug, deprecated path, hotfix window). Bias toward the orthodox engineering fix even when it costs more lines or touches more files.
2. If Quality Score is active: measure after fix, apply Keep/Discard rule, record in Experiment Ledger.
3. Repeat Steps 5-7.
4. **If same issue persists after 2 fix attempts**: Activate **Exploration Loop** (load `exploration-loop.md` per `context-loading.md`):
   - Generate 2-3 alternative approaches via Exploration Decision template
   - Re-spawn the same agent type with different hypothesis prompts (separate workspaces)
   - QA scores each result
   - Best result adopted, others discarded
   - All experiments recorded in Experiment Ledger
5. Continue until all critical issues are resolved.
6. Use memory write tool to record final results.
7. If Quality Score was measured: generate Experiment Ledger summary and auto-generate lessons from discarded experiments.

---

## Step 8: Optional Doc Verify Hook

To run docs verification after changes, load the `docs` skill and use `docs-verify`.
This hook is opt-in; skip by default.
5. If `docs` is not available (CLI command missing): skip silently.

This hook is opt-in; the default `auto_verify: false` skips this step entirely.
