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

## State Management: Universal File-First State I/O Architecture

All subagent coordination, deliverables, reviews, and progress tracking MUST follow the **Universal File-First State I/O Architecture**. Subagents write exhaustive deliverables directly to disk, avoiding conversational context dilution and maintaining an auditable orchestration trail.

### Universal Subagent Output Rule: File-First State I/O

1. **Exhaustive Artifact Writing**:
   - Every subagent task MUST write its full, detailed deliverables (code analysis, test evidence, architecture decisions, review matrices, checklists with line citations) to an explicit markdown file at:
     `.agents/results/{type}-{role}-{taskSlug}-{sessionId}[-{index}].md`
     - `{type}`: Artifact type (e.g., `result`, `progress`, `review`, `adr`, `test`, `benchmark`).
     - `{role}`: Agent role identifier (e.g., `backend`, `frontend`, `qa`, `reviewer`, `pm`, `designer`).
     - `{taskSlug}`: Concise kebab-case task identifier (e.g., `auth-jwt`, `cart-api`, `perf-audit`).
     - `{sessionId}`: Session identifier (e.g., `issue-104`, `conv-97e0b488`, `20260828-160543`).
     - `[-{index}]`: Optional numeric index when multiple artifacts or turns are produced.
   - Example: `.agents/results/result-backend-auth-jwt-issue-104.md` or `.agents/results/review-security-cart-api-20260828-160543-1.md`.

2. **Write Verification Before Chat Return**:
   - Subagents MUST verify that the deliverable file was successfully written and non-empty on disk before outputting their chat completion message.

3. **Standalone Fallback**:
   - If `OUTPUT_FILE` is not explicitly passed in the subagent prompt template, the subagent MUST auto-generate a timestamped destination:
     `.agents/results/result-{role}-{taskSlug}-$(date +%Y%m%d-%H%M%S).md`

4. **Universal 4-Line Chat Return Contract**:
   - Upon completing execution (whether SUCCESS, BLOCKED, or FAILED), subagents MUST return ONLY the concise 4-line standardized format in chat to conserve orchestrator context:
     ```markdown
     ### Task Complete: {Role} — {Task Name}
     - **Status**: SUCCESS | BLOCKED | FAILED
     - **Summary**:
       - {Key outcome, finding, decision, or change 1}
       - {Key outcome, finding, decision, or change 2}
       - {Key outcome, finding, decision, or change 3}
     - **Artifact**: `file:///path/to/{output-file}.md`
     ```

### Path Resolution (CRITICAL)

All result, progress, review, and state files MUST be written to the **project root** `.agents/results/` directory, never to a subdirectory or volatile temp directory.
- **Project root** = the git repository root (where `.git` exists).

## On Start

1. Confirm assigned task parameters: `SESSION_ID`, `TASK_SLUG`, and designated `OUTPUT_FILE` (or apply Standalone Fallback).
2. Read `.agents/results/task-board.md` (or parent task prompt) to confirm requirements and dependencies.
3. Pre-flight load `docs/checklists/<domain>.md` in host root (fallback: `.agents/skills/<skill>/resources/checklist.md`).
4. Initialize `.agents/results/progress-{role}-{taskSlug}-{sessionId}.md` with initial task status, plan, and target files.

## During Execution

- Periodically update `.agents/results/progress-{role}-{taskSlug}-{sessionId}.md` with active execution milestones.
- Include: actions taken, current phase/step status, files created/modified, and blockers.

## On Completion

1. Write exhaustive deliverables to the designated `OUTPUT_FILE` (`.agents/results/{type}-{role}-{taskSlug}-{sessionId}[-{index}].md`), containing:
   - Task metadata (Role, Task Slug, Session ID, Timestamp)
   - Status: `SUCCESS` or `FAILED`
   - Complete technical summary of work done
   - Full list of files created / modified / deleted
   - Acceptance criteria checklist with explicit line-number citations
   - Domain checklist verification evidence with explicit `file:line` citations
   - Test execution commands, outputs, and regression verification
2. Verify file exists on disk.
3. Return the standard 4-line chat return contract to the orchestrator/parent agent.

## On Failure / Blocked

1. Still write exhaustive diagnostic report to `OUTPUT_FILE` with Status: `FAILED` or `BLOCKED`.
2. Include full error stack traces, reproduction steps, root cause analysis, and remaining incomplete work.
3. Return the standard 4-line chat return contract with Status: `FAILED` or `BLOCKED` and artifact link.
