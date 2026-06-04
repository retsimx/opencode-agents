---
description: Structured bug diagnosis and fixing workflow that reproduces, diagnoses root cause, applies a minimal fix, writes regression tests, and scans for similar patterns
---

# MANDATORY RULES: VIOLATION IS FORBIDDEN

- **NEVER skip steps.** Execute from Step 1 in order.
- **Use Task subagents for isolated work** — delegate investigation subtasks (e.g. trace through a specific module) to subagents. Subagents are cheap; they prevent context dilution.
- **Use the `question` tool when uncertain** — never make assumptions about the bug. Ask if you need clarification.
- **You MUST use OpenCode's built-in tools for the workflow.**
  - Use `grep`, `glob`, `read` for bug investigation.
  - Use `write` and `edit` to record debugging results.
  - Use `.agents/results/` for output files.

---

### Subagent Spawn Criteria

Spawn `debug-investigator` when:
- Error spans multiple domains
- Similar pattern scan scope is 10+ files
- Deep dependency tracing is needed for diagnosis

### Spawning via OpenCode Task tool

When a subagent is needed (per criteria above), use the OpenCode Task tool:
- `subagent_type="general"` for debug investigation
- Include diagnosis results and scan scope in the prompt

---

## Step 1: Collect Error Information

Ask the user for:
- Error message, steps to reproduce
- Expected vs actual behavior
- Environment (browser, OS, device)

If an error message is provided, proceed immediately.

---

## Step 2: Reproduce the Bug

// turbo
Use `grep` with the error message or stack trace to locate the error in the codebase.

---

## Step 3: Diagnose Root Cause

Use `grep` to trace the execution path backward from the error point.
Identify the root cause, not just the symptom. Check:
- null/undefined access
- Race conditions
- Missing error handling
- Wrong data types
- Stale state

---

## Step 4: Propose Minimal Fix

Present the root cause and proposed fix to the user.
- The fix should change only what is necessary.
- Explain why this fixes the root cause, not just the symptom.
- **You MUST get user confirmation before proceeding to Step 5.**

---

## Step 5: Apply Fix and Write Regression Test

// turbo
1. Implement the minimal fix.
2. Write a regression test that reproduces the original bug and verifies the fix.
3. The test must fail without the fix and pass with it.

---

## Step 6: Scan for Similar Patterns

// turbo
Use `grep` to search the codebase for the same pattern that caused the bug.
Report any other locations that may have the same vulnerability. Fix them if confirmed.

---

## Step 7: Document the Bug

Write a bug report to `.agents/results/bug-report-{bug-id}.md`:
- Symptom, root cause
- Fix applied, files changed
- Regression test location
- Similar patterns found
