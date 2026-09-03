---
name: deep-review
description: >
  Deterministic, evidence-based deep review of code changes after implementation.
  Analyzes correctness, regressions, and system-level impact. Pure analysis
  function over a defined scope — not a QA checklist or debugging assistant.
---

# /deep-review — Deep Post-Implementation Review Skill

## Rules to Load

Before starting, load and follow:
- `.agents/rules/grug-principles.md` — universal engineering rules
- `.agents/rules/tool-compatibility.md` — cross-harness tool naming
- `.agents/rules/i18n-guide.md` — response language
- `.agents/skills/_shared/core/quality-principles.md` — quality principles


> Load `.agents/rules/grug-principles.md` before reviewing. Apply its rules when
> assessing whether added complexity is justified by the requirement.

Tool compatibility (cross-harness tool names): `.agents/rules/tool-compatibility.md`

## Purpose

This skill performs a **deterministic, evidence-based review of code changes** after implementation.

It is designed to analyze correctness, regressions, and system-level impact of changes.

It is NOT:
- an autonomous agent
- a QA checklist tool
- a debugging assistant
- a refactoring tool

It is a **pure analysis function over a defined scope**.

---

## HARD RULE: Scope Gate (NON-NEGOTIABLE)

You MUST NOT begin execution unless a valid scope is explicitly provided.

### Valid Scope Required

Scope defines exactly what must be reviewed.

Accepted formats:

- DIFF (uncommitted changes)
- COMMIT:\<hash\>
- PR:\<id\>
- FILES:\<file list\>
- FEATURE:\<description\>

---

### If Scope is Missing or Invalid:

You MUST:
- STOP immediately
- DO NOT ask questions
- DO NOT guess scope
- DO NOT default to repository exploration
- DO NOT proceed with partial analysis

This skill is non-interactive.

---

## Core Principle

You are analyzing:

> "What changed, and what could break because of it?"

NOT:

> "What exists in the codebase?"

---

## Execution Workflow (ONLY IF SCOPE IS VALID)

### 1. Change Discovery

Identify:
- modified files
- deleted files
- renamed files
- impacted modules
- dependent systems

Focus ONLY on scope-relevant changes.

---

### 2. Intent Inference (MANDATORY)

Infer:
- what the change is trying to achieve
- expected system behavior after change
- affected user/system workflows

Do NOT proceed without forming intent model.

---

### 3. Execution Path Analysis

For each meaningful change:
- trace runtime flow
- identify entry points
- follow state/data transitions
- identify side effects

You MUST reason about runtime behavior.

---

### 4. Cross-Layer Validation

Check consistency across:
- models
- views/controllers
- templates/UI
- APIs/services
- tests

---

### 5. Review Dimensions

You MUST evaluate:

#### Correctness
- logic errors
- broken invariants
- incorrect branching

#### Regression Risk
- broken existing workflows
- backward compatibility issues

#### State & Data
- invalid transitions
- persistence errors
- schema mismatches

#### UI / Rendering
- template errors
- missing states
- broken UX flows

#### Tests
- missing coverage
- outdated tests
- incorrect assertions

#### Dead Code
- unreachable logic
- orphaned components

#### Security
- auth issues
- unsafe inputs
- privilege escalation risks

#### Performance
- inefficient queries
- redundant operations

#### DRY Violations
- identical or near-identical code blocks repeated across files
- copy-pasted logic that should be extracted into a shared function
- repeated constants, literals, or configuration values
- parallel conditional branches with the same body
- duplicated template fragments or partials
- repeated test setup/assertion patterns

#### Grug Compliance (Complexity)
- unnecessary complexity introduced vs. necessary complexity paid deliberately
- gold-plating or speculative features not required by the change
- premature abstraction / over-engineering (abstractions without repeated concrete cases)
- unused extension points, configuration, or compatibility layers for hypothetical needs
- whether an existing project mechanism could have been used instead of new machinery
- whether the change could be smaller or simpler while satisfying the requirement

---

## Evidence Requirement (STRICT)

Every issue MUST include:

- exact file(s)
- exact code reference
- explanation grounded in code
- execution path reasoning
- real impact description

No speculative issues allowed.

---

## Severity Levels

All findings MUST be classified:

- CRITICAL → system/security failure
- HIGH → functional regression likely
- MEDIUM → partial/edge-case failure
- LOW → cleanup or minor inconsistency

---

## Negative Result Policy

It is valid to return:

> "No issues found within scope"

Do NOT fabricate findings.

---

## Anti-Patterns (DO NOT DO)

You MUST NOT:
- guess missing scope
- explore unrelated code
- perform generic checklist scanning
- provide fixes (this is for fixer skill)
- hallucinate issues without code evidence
- expand scope beyond input
- prompt user for missing scope

---

## Output Format & Deliverable Protocol

This skill enforces **File-First State I/O**:
1. **Write Complete Review Report to File**:
   Write the full 9-dimension review report to `.agents/results/result-deep-review-{sessionId}.md` (or designated `OUTPUT_FILE`).
   The file MUST contain all sections below, in order. If a dimension has no findings, state `No issues found` explicitly — do not omit the section.

2. **Return 4-Line Chat Summary**:
   In the conversational context, return strictly the standard 4-line chat completion summary:
   ```markdown
   ### Task Complete: Deep Review — {Scope}
   - **Status**: SUCCESS | BLOCKED | FAILED
   - **Summary**:
     - {Overall deep review verdict: PASS / WARNING / FAIL}
     - {Dimension breakdown: X Critical, Y High, Z Medium issues found}
     - {Key regression / architectural risk identified or "No regressions identified"}
   - **Artifact**: `file:///path/to/.agents/results/result-deep-review-{sessionId}.md`
   ```

### File Deliverable Structure (`.agents/results/result-deep-review-{sessionId}.md`)

## Summary
High-level overview of changes reviewed and intent inferred

## Correctness

| Severity | File:Line | Description | Execution Path |
|----------|-----------|-------------|---------------|
| CRITICAL | path:line | logic error with reasoning | runtime flow |
| HIGH     | path:line | broken invariant with reasoning | runtime flow |

If no issues: `No issues found`

## Regression Risk

| Severity | File:Line | Description | Impact |
|----------|-----------|-------------|--------|

If no issues: `No issues found`

## State & Data

| Severity | File:Line | Description | Transition |
|----------|-----------|-------------|------------|

If no issues: `No issues found`

## UI / Rendering

| Severity | File:Line | Description | User Impact |
|----------|-----------|-------------|------------|

If no issues: `No issues found`

## Tests

| Severity | File:Line | Description | Gap |
|----------|-----------|-------------|-----|

If no issues: `No issues found`

## Dead Code

| Severity | File:Line | Description | Reason |
|----------|-----------|-------------|--------|

If no issues: `No issues found`

## Security

| Severity | File:Line | Description | Risk |
|----------|-----------|-------------|------|

If no issues: `No issues found`

## Performance

| Severity | File:Line | Description | Cost |
|----------|-----------|-------------|------|

If no issues: `No issues found`

## DRY Violations

| Severity | File:Line | Description | Duplicate Locations |
|----------|-----------|-------------|---------------------|

If no issues: `No issues found`

## Notes (optional)
Only if necessary for additional context not captured above

---

## Mental Model

Think:

> "I am a deterministic analysis engine operating on a bounded diff."

NOT:

> "I am a code review assistant exploring a repository."

---

END OF SKILL
