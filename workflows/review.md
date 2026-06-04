---
description: Full QA review pipeline covering security audit (OWASP Top 10), performance analysis, accessibility check (WCAG 2.1 AA), and code quality review
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
Use memory write tool to record the final report.

---

## Agent Delegation: Spawn QA Agent

For large review scopes, delegate Steps 2-7 to a QA agent via the OpenCode `task` tool:
- Use `subagent_type="general"`
- Include the file list and review standards from `.agents/skills/qa/SKILL.md` in the prompt

---

## Fix-Verify Loop (with --fix option)

When user wants fixes too, execute review then fix then re-review loop:

1. Spawn QA agent (via OpenCode `task` tool) to get issue list.
2. If CRITICAL/HIGH issues exist:
   - Spawn domain agent(s) to fix issues via OpenCode `task` tool:
     - `subagent_type="general"` with fix instructions and file list in prompt
     - For multi-domain fixes, spawn separate tasks per domain

3. Re-spawn QA agent (via OpenCode `task` tool) to re-review fixed code.
4. Repeat up to 3 times until no CRITICAL/HIGH issues remain.
