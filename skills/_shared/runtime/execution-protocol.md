# Execution Protocol (Vendor-Agnostic)

When spawned via the OpenCode `task` tool or subagent delegation, follow this protocol for shared state coordination and rigorous execution.

---

## Key Practices

- **Pre-flight Tier 0 Checklist Injection** — Before starting implementation, always load `docs/checklists/<domain>.md` (fallback: `.agents/skills/<skill>/resources/checklist.md`).
- **Use Task subagents for isolated work** — delegate distinct subtasks to sub-subagents rather than doing everything inline. Each subagent gets a focused context, reducing dilution and preventing scope creep.
- **Ask when uncertain** — use the `question` tool rather than making assumptions. Guessing leads to wasted work.
- **Stay in scope** — only work on your assigned task. Flag out-of-scope issues without fixing them.

---

## Core Execution Steps

### Step 0: Prepare & Pre-Flight
1. **Assess difficulty**: see `.agents/skills/_shared/core/difficulty-guide.md`
2. **Pre-flight checklist injection**: load `docs/checklists/<domain>.md` in the host project root (fallback: `.agents/skills/<skill>/resources/checklist.md`).
3. **Check bug post-mortems**: review relevant incident post-mortems in `.agents/results/bugs/` if working in high-risk or previously defected modules.
4. **Clarify requirements**: follow `.agents/skills/_shared/core/clarification-protocol.md`.

### Step 1: Analyze
- Read requirements carefully.
- Search codebase (`grep`/`glob`) to locate touch points and verify assumptions.
- Identify dependencies and boundary constraints.

### Step 2: Plan
- Define architecture, schemas, and contract interfaces.
- Specify regression tests and edge cases.

### Step 3: Implement
- Execute minimal, clean code changes matching task requirements.
- Write unit/integration and regression tests covering changed behavior.

### Step 4: Verify & Checklists (Line-Number Citations Required)
- **Checklist Verification with Explicit Citations**: When verifying against `docs/checklists/<domain>.md` and skill-local checklists, each checklist item MUST include explicit `file:line` citations showing where the rule was adhered to in the codebase.
  - *Example*: `- [x] Timezone: Verified src/api/views.py:L45 uses timezone.localdate()`
  - *Example*: `- [x] CSRF Protection: Verified templates/form.html:L12 includes {% csrf_token %}`
- **Run automated test suites and linters**: ensure all tests pass with zero regressions.
- **Runtime verification**: confirm behavior in the runtime environment.

---

## Bug Closure Gate

Whenever `debug-agent` or any agent finishes investigating and resolving a bug:
1. **Save Post-Mortem**: Document root cause, reproduction, and fix in `.agents/results/bugs/` (using bug report template).
2. **Extract 1-Line Guardrail**: Extract a concise 1-line operational rule into the relevant domain checklist (`docs/checklists/<domain>.md`).
   - The rule must strictly conform to the standardized regex pattern:
     ```markdown
     - [ ] **{Topic}**: {Rule} (❌ Anti-pattern: `{X}`, ✅ Required: `{Y}`) [Ref: .agents/results/bugs/{incident-file}.md]
     ```
   - *Conforming Example*:
     `- [ ] **Timezone Local Date**: Use timezone.localdate() instead of timezone.now().date() or date.today() for date comparisons, weekday resolution, and calendar boundaries. (❌ Anti-pattern: `today = timezone.now().date()`, ✅ Required: `today = timezone.localdate()`) [Ref: .agents/results/bugs/issue-123-example-incident.md]`
3. **Enforce Gate**: A bug resolution is NOT complete until the 1-line rule is committed to `docs/checklists/<domain>.md`.

---

## 3-State Graduation Lifecycle

Guardrails and operational lessons follow a strict 3-state progression to maintain a high-signal, non-bloated checklist:

```
┌────────────────────────────────┐
│ 1. Incident (Post-Mortem)      │  Deep RCA artifact saved in .agents/results/bugs/
└──────────────┬─────────────────┘
               │  Bug Closure Gate: Extract 1-line rule
               ▼
┌────────────────────────────────┐
│ 2. Active Checklist            │  1-line operational guardrail in docs/checklists/<domain>.md
│    (Operational Guardrail)     │  Injected at Tier 0 pre-flight; verified in Step 4 with L<line> citations
└──────────────┬─────────────────┘
               │  Graduation: Automated via Linter or CI Test
               ▼
┌────────────────────────────────┐
│ 3. Graduated to Linter/Test    │  Enforced automatically by linter/analyzer or automated test suite.
│    (Automated Rule)            │  Retired from manual checklist once automated.
└────────────────────────────────┘
```

1. **State 1: Incident (Post-Mortem)**
   - Triggered by bugs, regressions, or high Clarification Debt (CD >= 50).
   - Detailed RCA recorded in `.agents/results/bugs/` containing root causes, reproduction steps, and architectural analysis.
2. **State 2: Active Checklist (Operational Guardrail)**
   - Actionable 1-line guardrail active in `docs/checklists/<domain>.md`.
   - Injected pre-flight for all domain agents and verified with explicit line citations in Step 4.
3. **State 3: Graduated to Linter/Test (Automated Rule)**
   - When a rule is codified into a deterministic tool (e.g. Ruff rule, ESLint plugin, AST check, custom CI assertion, or unit test), it graduates out of the manual checklist.
   - Removed from `docs/checklists/<domain>.md` to avoid checklist bloat while maintaining automated enforcement.

---

## State Management

Use file-based I/O for coordination. Write results to `.agents/results/`.

### Path Resolution (CRITICAL)

All result, progress, and state files MUST be written to the **project root** `.agents/` directory, never to a subdirectory's `.agents/`.

- **Project root** = the git repository root (where `.git` exists)
- **Session-scoped naming**: when running under an orchestration session, append session ID as suffix:
  - `result-{agent-id}-{sessionId}.md` (e.g., `result-frontend-session-20260405-100835.md`)
  - `progress-{agent-id}-{sessionId}.md`
- **Manual (non-orchestrated) runs**: no suffix, `result-{agent-id}.md`

## On Start

1. Read `.agents/results/task-board.md` to confirm your assigned task
2. Pre-flight load `docs/checklists/<domain>.md`
3. Create `.agents/results/progress-{agent-id}[-{sessionId}].md` with initial status

## During Execution

- Periodically update `progress-{agent-id}[-{sessionId}].md` with current state
- Include: action taken, current status, files created/modified

## On Completion

- Create `.agents/results/result-{agent-id}[-{sessionId}].md` with final result including:
  - Status: `completed` or `failed`
  - Summary of work done
  - Files created/modified
  - Acceptance criteria checklist with line-number citations
  - Domain checklist verification evidence with line-number citations

## On Failure

- Still create `result-{agent-id}[-{sessionId}].md` with Status: `failed`
- Include detailed error description and what remains incomplete
