---
name: review
description: >
  Full QA review pipeline covering security audit (OWASP Top 10), performance
  analysis, accessibility checks (WCAG 2.1 AA), and code quality review.
  Use for pre-ship code review, PR review, branch review, or security audit
  of any codebase.
---

# QA Review Pipeline (`review`)

## Scheduling

### Goal
Run a comprehensive review of code changes: automated security scanning, manual OWASP Top 10 review, performance analysis, accessibility checks, code quality review, and a prioritized findings report.

### Intent signature
- User says "review", "audit", "QA", "security review", "check code quality"
- User wants a pre-ship review or PR review
- User asks about OWASP, vulnerabilities, performance issues, or accessibility

### When to use
Use before shipping any changes, after receiving code from implementation agents, or on explicit request. Can delegate to a QA subagent for large scopes.

## Structural Flow

Execute `.agents/workflows/review.md` which covers the full 7-step pipeline: scope identification, automated security scanning, OWASP manual review, performance analysis, accessibility review, code quality review, and QA report generation with fix-verify loop.

## Logical Operations

All logic is defined in the workflow file at `.agents/workflows/review.md`. Load and follow it step-by-step without deviation.

## References

- Workflow: `.agents/workflows/review.md`
- QA skill: `.agents/skills/qa/SKILL.md`
- Coordination skill: `.agents/skills/coordination/SKILL.md`
