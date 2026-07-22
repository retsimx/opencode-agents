---
name: plan
description: Product-management planning workflow that turns ambiguous or complex product requests into actionable, dependency-aware task breakdowns. Gathers requirements, defines API contracts, decomposes work into prioritized tasks with acceptance criteria, and produces both a machine-readable plan and a human-readable tracker in docs/plans/. Use for planning, requirements, specification, scope, prioritization, task breakdown, roadmap, implementation plan, and ISO 21500 / ISO 31000 / ISO 38500-aligned planning recommendations.
---

# MANDATORY RULES: VIOLATION IS FORBIDDEN

- **NEVER skip steps.** Execute from Step 1 in order.
- **Use Task subagents for isolated work** — delegate in-depth analysis of specific areas to exploration subagents. Subagents are cheap; they prevent context dilution.
- **Use the `question` tool when uncertain** — never make assumptions about requirements. Ask if you need clarification.
- **You MUST use OpenCode's built-in tools for the workflow.**
  - Use `grep`, `glob`, `read` for codebase exploration.
  - Use `write` and `edit` to record planning results.
  - Use `.agents/results/` for output files.

---

> **Vendor note:** This workflow executes inline (no subagent spawning). All vendors use their native code analysis tools. Plan artifacts (`.agents/results/plan-{sessionId}.json` and `docs/plans/work/{NNN}-{name}.md`) are consumed by the orchestrate or work skills, which handle their own vendor detection.

---

## Core Philosophy

**Plans are first-class artifacts**: structured, templated, and consumed by other skills. They are local working artifacts (not committed to the repo; `docs/plans/` is gitignored), but they follow strict conventions so any agent can read and update them.

Two artifacts per plan:

1. **Machine-readable**: `.agents/results/plan-{sessionId}.json` consumed by orchestrate and work.
2. **Human-readable**: `docs/plans/work/{NNN}-{name}.md` with task table, decision log, and progress notes. Lifecycle is tracked via the `Status` field in the file header (`Active` → `Completed`); no folder moves required.

### Layout

```
docs/plans/
├── designs/                       ← permanent design references (Status: Approved/Draft/Superseded)
│   └── {NNN}-{name}.md
└── work/                          ← execution plans (Status: Active/Completed)
    ├── {NNN}-{name}.md
    └── tech-debt-tracker.md
```

- Folder = type (designs vs work). Status field = lifecycle.
- Filename always uses 3-digit zero-padded sequential prefix (`001-`, `002-`, …) per folder.
- Determine the next number with `ls docs/plans/{designs,work}/ | grep -E '^[0-9]{3}-' | tail -1`.

---

## When to use

- Breaking down complex feature requests into tasks
- Determining technical feasibility and architecture sequencing
- Prioritizing work and planning sprints or phases
- Defining API contracts and data models before implementation
- Standards-aligned planning (ISO 21500 scope/dependency framing, ISO 31000 risk treatments, ISO 38500 governance/decision rights)

## When NOT to use

- Implementing actual code → delegate to backend / frontend / mobile / db
- Performing pre-ship code review → use `review`
- Diagnosing a specific bug → use `debug`

---

## Step 1: Gather Requirements

Ask the user to describe what they want to build. Clarify:
- Target users
- Core features (must-have vs nice-to-have)
- Constraints (tech stack, existing codebase)
- Deployment target (web, mobile, both)

Follow `.agents/skills/_shared/core/clarification-protocol.md` — check Uncertainty Triggers (business logic, security/auth, existing code conflicts). LOW → proceed, MEDIUM → present options, HIGH → ask immediately.

---

## Step 2: Analyze Technical Feasibility

// turbo
If an existing codebase exists:
- Use `grep`/`glob` to analyze the codebase.

Also search `docs/plans/work/` for related past or in-progress plans, and `docs/plans/designs/` for prior design references. Reuse patterns from similar work.

Read cross-domain entries in `.agents/skills/_shared/core/lessons-learned.md` to avoid known traps.

---

## Step 3: Assess Complexity

Use the difficulty-guide skill to classify:

- **Simple** → no plan artifact needed; execute directly via work skill.
- **Medium** → produce both JSON and a lightweight markdown tracker (skip Step 4 API contracts if not cross-boundary).
- **Complex** → produce both artifacts with all sections plus API contracts.

Report scope assessment to the user. Get confirmation before proceeding.

---

## Step 4: Define API Contracts

// turbo
If the plan involves cross-boundary work (frontend ↔ backend, service ↔ service):

1. Design API contracts using the api-contracts template. Per endpoint:
   - Method, path, request/response schemas
   - Auth requirements, error responses
2. Save to `api-contracts/{contract-name}.md`.
3. Reference from the markdown tracker generated in Step 6.

API-first design: contracts are defined **before** implementation tasks. This unblocks parallel frontend/backend execution.

---

## Step 5: Decompose into Tasks

// turbo
Break down the project into actionable tasks. Each task must have:
- **Assigned agent** (backend / frontend / mobile / db / review / debug)
- **Title** and short description
- **Acceptance criteria** (testable, measurable)
- **Priority** (P0 independent, P1 depends on P0, etc.)
- **Dependencies** (task IDs)
- **Scope**: array of directory prefixes the agent is allowed to modify (e.g., `["src/api/", "migrations/"]`). Used by `verify` to detect cross-agent boundary violations in parallel execution.

**Engineering-first decomposition:** prefer tasks that address root causes over tasks that patch individual symptoms. When a deliberate workaround or hotfix is included, record the reason in the Decision Log.

**Sizing:** each task should be completable by a single agent in 10-20 turns. Merge tasks < 5 turns; split tasks > 30 turns.

Security and testing are part of every task — not separate phases.

---

## Step 6: Review Plan with User

Present the full plan: task list, priority tiers, dependency graph, agent assignments, completion criteria.
**You MUST get user confirmation before proceeding to Step 7.**

---

## Step 7: Save Plan Artifacts

// turbo
Generate both artifacts.

### 7a. Machine-readable plan

Save `.agents/results/plan-{sessionId}.json` (and `.agents/results/result-plan.md` for human consumption). The JSON shape follows `.agents/skills/plan/resources/task-template.json`. Write a memory summary via the configured memory tool.

### 7b. Human-readable tracker (Medium/Complex only)

Generate `docs/plans/work/{NNN}-{name}.md` using this template (replace `{NNN}` with the next zero-padded 3-digit number for the `work/` folder):

```markdown
# {Plan Title}

> {One-line goal}

**Status**: Active
**Created**: {date}
**Owner**: {agent or human}

## Goal
{What this plan achieves — clear, testable outcome}

## Context
{Relevant background, related code, prior decisions}

## Constraints
{Rules, dependencies, compatibility requirements}

## Tasks

| # | Task | Agent | Priority | Status | Dependencies |
|---|------|-------|----------|--------|--------------|
| 1 | {task} | {agent} | P0 | TODO | — |
| 2 | {task} | {agent} | P0 | TODO | 1 |
| 3 | {task} | {agent} | P1 | TODO | 1, 2 |

## Done When
{Testable completion criteria}
- [ ] {criterion 1}
- [ ] {criterion 2}

## Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| {date} | {what was decided} | {why} |

## Progress Notes
{Append-only log of progress updates}

- [{date}] Plan created
```

### Naming Convention

- Format: `{NNN}-{kebab-name}.md` (e.g., `008-add-user-authentication.md`).
- `{NNN}` is the next zero-padded 3-digit sequential number for that folder. Determine it from the existing files: `ls docs/plans/work/ | grep -E '^[0-9]{3}-' | tail -1`.
- `{kebab-name}` describes the feature; do **not** append `-design` or `-plan` (the folder already encodes type).
- Lifecycle is tracked via the `Status` header in the file, not via folder moves.

The plan is now ready for the work or orchestrate skills to execute.

---

## Lifecycle Updates (during execution)

The orchestrate and work skills update the markdown tracker as work progresses:

- Task status: `TODO` → `WIP` → `DONE` or `BLOCKED`
- Append timestamped entries to **Progress Notes**
- Record cross-cutting decisions in the **Decision Log**

When all "Done When" criteria are met:

1. Set the header `Status` field: `Active` → `Completed`.
2. Append a completion summary to Progress Notes with the date.
3. The file stays in `docs/plans/work/`; no move required.
4. If any tech debt was introduced, update `docs/plans/work/tech-debt-tracker.md`.

To list in-progress plans: `grep -l "^\*\*Status\*\*: Active" docs/plans/work/*.md`.

---

## Tech Debt Tracker

`docs/plans/work/tech-debt-tracker.md` tracks known debt across all plans:

```markdown
# Tech Debt Tracker

| # | Debt | Source Plan | Priority | Proposed Resolution |
|---|------|-------------|----------|---------------------|
| 1 | {description} | {plan-name} | P1 | {how to fix} |
```

- Add entries when shortcuts are taken during plan execution.
- Remove entries when debt is resolved.
- Review periodically; debt items can become plans themselves.

---

## Standards-Aligned Planning (Optional)

When the user asks for enterprise delivery structure, risk-aware prioritization, or governance-oriented planning, read `.agents/skills/plan/resources/iso-planning.md` and add a **Standards-Aligned Planning Notes** section to the tracker:

```md
## Standards-Aligned Planning Notes

### ISO 21500
- Scope:
- Stakeholders:
- Major dependencies:

### ISO 31000
- Top risks:
- Mitigations:
- Residual risks:

### ISO 38500
- Decision owner:
- Required approvals:
- Governance checkpoints:
```

Apply this lens when the project is enterprise/regulated, many stakeholders are involved, risk is material, or decision rights are unclear. Avoid the overhead on small features or bug fixes.

---

## Guardrails

1. **API-first design** — define contracts before implementation tasks.
2. Every task has: agent, title, acceptance criteria, priority, dependencies, scope.
3. **Minimize dependencies** for maximum parallel execution.
4. **Security and testing are part of every task** (not separate phases).
5. Tasks should be completable by a single agent.
6. Output JSON plan + markdown tracker for orchestrate compatibility.
7. When relevant, structure plans using ISO 21500 concepts, prioritize risk using ISO 31000 thinking, and surface responsibility/governance suggestions inspired by ISO 38500.

## Common Pitfalls

- **Too granular**: "Implement user auth API" is one task, not five.
- **Vague tasks**: "Make it better" → "Add loading states to all forms".
- **Tight coupling**: tasks should use public APIs, not internal state.
- **Deferred quality**: testing is part of every task, not a final phase.

---

## References

- Task schema: `.agents/skills/plan/resources/task-template.json`
- Plan examples: `.agents/skills/plan/resources/examples.md`
- Execution protocol: `.agents/skills/plan/resources/execution-protocol.md`
- ISO planning guide: `.agents/skills/plan/resources/iso-planning.md`
- Error recovery: `.agents/skills/plan/resources/error-playbook.md`
- API contracts: `.agents/skills/_shared/core/api-contracts/`
- Context loading: `.agents/skills/_shared/core/context-loading.md`
- Difficulty guide: `.agents/skills/_shared/core/difficulty-guide.md`
- Clarification protocol: `.agents/skills/_shared/core/clarification-protocol.md`
- Reasoning templates: `.agents/skills/_shared/core/reasoning-templates.md`
- Lessons learned: `.agents/skills/_shared/core/lessons-learned.md`
