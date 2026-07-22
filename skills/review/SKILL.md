---
name: review
description: Full QA review pipeline covering security audit (OWASP Top 10), performance analysis, accessibility checks (WCAG 2.1 AA), and code quality review. Use for pre-ship code review, PR review, branch review, security audit, accessibility audit, performance review, dependency CVE scan, lint-driven review, and ISO/IEC 25010 quality model / ISO/IEC 29119 test-process alignment.
---

# MANDATORY RULES: VIOLATION IS FORBIDDEN

- **NEVER skip steps.** Execute from Step 1 in order.
- **Use Task subagents for isolated work** — delegate review focus areas (security, performance, etc.) to separate review subagents. Subagents are cheap; they prevent context dilution.
- **Use the `question` tool when uncertain** — never make assumptions about code intent. Ask if you need clarification.
- **You MUST use OpenCode's built-in tools for the workflow.**
  - Use `grep`, `glob`, `read` for code analysis and review.
  - Use `write` and `edit` to record review results.
  - Use `.agents/results/` for output files.

---

## When to use

- Pre-ship code review, PR review, branch review
- Security audit (OWASP Top 10, dependency CVEs)
- Performance analysis and benchmarking
- Accessibility review (WCAG 2.1 AA, ARIA, keyboard nav)
- Code quality (lint, type safety, naming, async/await, API docs)
- Standards-aligned quality framing (ISO/IEC 25010 quality model, ISO/IEC 29119 test process)

## When NOT to use

- Pure planning, scope, or task breakdown → use `plan`
- Architecture decisions, ADRs, system-design choices → use `architecture`
- Diagnosing a single bug with a stack trace or repro → use `debug`
- Specialized deep security research (threat model, exploit chains) → use `deepsec`
- Visual / interaction / motion design review → use `design`

---

## Step 1: Identify Review Scope

Ask the user what to review: specific files, a feature branch, or the entire project.
If a PR or branch is provided, diff against the base branch to scope the review.

---

## Step 2: Run Automated Security Checks

// turbo
Run available security tools: `npm audit` (Node.js), `bandit` (Python), or equivalent.
Check for known vulnerabilities in dependencies. Flag any CRITICAL or HIGH findings.

---

## Step 3: Manual Security Review (OWASP Top 10)

Use `grep` and `read` to review code for:
- Injection (SQL, XSS, command)
- Broken auth, sensitive data exposure
- Broken access control, security misconfig
- Insecure deserialization
- Known vulnerable components
- Insufficient logging

---

## Step 4: Performance Analysis

Use `grep` to check for:
- N+1 queries, missing indexes
- Unbounded pagination, memory leaks
- Unnecessary re-renders (React)
- Missing lazy loading
- Large bundle sizes, unoptimized images

---

## Step 5: Accessibility Review (WCAG 2.1 AA)

Check for:
- Semantic HTML, ARIA labels
- Keyboard navigation, color contrast
- Focus management, screen reader compatibility
- Image alt text

---

## Step 6: Code Quality Review

Use `grep`/`glob` and `read` to check for:
- Consistent naming, proper error handling
- Test coverage, TypeScript strict mode compliance
- Unused imports/variables
- Proper async/await usage
- Public API documentation

---

## Step 7: Generate QA Report

Compile all findings into a prioritized report:
- **CRITICAL**: Security breaches, data loss risks
- **HIGH**: Blocks launch
- **MEDIUM**: Fix this sprint
- **LOW**: Backlog

Each finding must include: `file:line`, description, and remediation code.
Write the final report to `.agents/results/` per `.agents/skills/_shared/runtime/coordination-protocol.md`.

---

## Agent Delegation: Spawn Review Agent

For large review scopes, delegate Steps 2-7 to a review agent via the OpenCode `task` tool:
- Use `subagent_type="general"`
- Include the file list and review standards from the review skill in the prompt

---

## Fix-Verify Loop (with --fix option)

When user wants fixes too, execute review then fix then re-review loop:

1. Spawn review agent (via OpenCode `task` tool) to get issue list.
2. If CRITICAL/HIGH issues exist:
   - Spawn domain agent(s) to fix issues via OpenCode `task` tool:
     - `subagent_type="general"` with fix instructions and file list in prompt
     - For multi-domain fixes, spawn separate tasks per domain

3. Re-spawn review agent (via OpenCode `task` tool) to re-review fixed code.
4. Repeat up to 3 times until no CRITICAL/HIGH issues remain.

---

## Standards-Aligned Review (Optional)

When the user asks for an enterprise or compliance framing, or when the project is regulated, read `.agents/skills/review/resources/iso-quality.md` and frame findings against the ISO/IEC 25010 quality model and ISO/IEC 29119 test process. Surface:

- **Quality model (ISO/IEC 25010)**: functional suitability, performance efficiency, compatibility, usability, reliability, security, maintainability, portability.
- **Test process (ISO/IEC 29119)**: org test strategy, dynamic/functional test design, test execution, reporting.
- **Verdict per characteristic**: acceptable / needs work / not measured.

Add this section to the report when relevant; do not bloat small reviews with the overhead.

---

## Self-Check Before Reporting

Before finalizing the report, walk `.agents/skills/review/resources/self-check.md` to confirm:

- Findings cite `file:line` and include remediation code or diff.
- Severity tier is justified (CRITICAL/HIGH/MEDIUM/LOW).
- The "what we did NOT review" list is explicit (e.g., unauthenticated infrastructure, third-party APIs).
- Re-review / re-scan commands are listed for the user.

---

## Common Pitfalls

- **Vague findings**: "looks risky" → cite `file:line`, describe impact, give a fix.
- **Auto-tool bias**: `npm audit` finds dependency CVEs; it does not catch authz logic errors. Combine automated and manual.
- **Severity inflation**: not every issue is CRITICAL. Reserve CRITICAL for security breaches / data loss.
- **No remediation**: a finding without a fix is half a report.

---

## References

- Review checklist: `.agents/skills/review/resources/checklist.md`
- Self-check before report: `.agents/skills/review/resources/self-check.md`
- Report examples: `.agents/skills/review/resources/examples.md`
- Execution protocol: `.agents/skills/review/resources/execution-protocol.md`
- ISO/IEC quality guide: `.agents/skills/review/resources/iso-quality.md`
- Error recovery: `.agents/skills/review/resources/error-playbook.md`
- Lessons learned: `.agents/skills/_shared/core/lessons-learned.md`
- Clarification protocol: `.agents/skills/_shared/core/clarification-protocol.md`
- Reasoning templates: `.agents/skills/_shared/core/reasoning-templates.md`
