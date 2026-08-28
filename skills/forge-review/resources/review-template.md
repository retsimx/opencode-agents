# Forge Review: Top-Level PR/MR Review Summary Template

This resource defines the standardized schema for the top-level review summary posted to GitHub Pull Requests or GitLab Merge Requests by `forge-review`.

---

## Top-Level Review Markdown Template

```markdown
# Code Review Summary: PR #{PR_NUMBER} — {PR_TITLE}

**Review Verdict**: `{VERDICT}` <!-- Options: APPROVE | REQUEST_CHANGES | COMMENT -->
**Evaluated Scope**: `{COMMIT_RANGE}` (`{HEAD_SHA}`)
**Target Branch**: `{BASE_BRANCH}`
**Review Date**: `{DATE_TIME_UTC}`

---

## Executive Summary

{2-4 sentence executive overview summarizing the scope of changes, overall code quality, acceptance criteria compliance, and critical findings requiring author action.}

---

## 1. Acceptance Criteria & Contract Alignment Matrix

This matrix verifies that all requirements from the associated Issue(s) and Epic specifications are fully implemented and verified with concrete code citations.

| # | Requirement / Acceptance Criterion | Source Ref | Implementation Status | Evidence / Code Proof (`file:line`) | Notes / Gaps |
|---|-------------------------------------|------------|-----------------------|--------------------------------------|--------------|
| 1 | {Requirement description 1} | Issue #{NUM} / Epic #{NUM} | `VERIFIED` | `path/to/file.py:L45-L52` | {Implementation details or verification note} |
| 2 | {Requirement description 2} | Issue #{NUM} | `VERIFIED` | `path/to/view.py:L102` | {Tested in test_suite.py:L33} |
| 3 | {Requirement description 3} | Issue #{NUM} | `INCOMPLETE` / `DEVIATED` / `MISSING` | `path/to/handler.py:L80` | {Specific missing edge case or deviation} |

> **Status Legend**:
> - `VERIFIED`: Requirement is completely implemented, covered by tests, and adheres to contract.
> - `INCOMPLETE`: Partially implemented; secondary edge cases or parameters missing.
> - `DEVIATED`: Implemented differently than specified in the issue/epic contract without documented rationale.
> - `MISSING`: Requirement was specified in the issue/epic but has no implementation in the diff.

---

## 2. 9-Dimension Quality & Security Audit

Comprehensive evaluation across the 9 core software engineering dimensions:

| # | Dimension | Status | Findings Summary |
|---|-----------|:------:|------------------|
| 1 | **Correctness** | `PASS` / `WARN` / `FAIL` | {Summary of logic errors, invariants, or edge-case handling} |
| 2 | **Security & Auth** | `PASS` / `WARN` / `FAIL` | {Summary of auth, authorization/IDOR, injection, or data leaks} |
| 3 | **Regression Risk** | `PASS` / `WARN` / `FAIL` | {Summary of backward compatibility or broken existing workflows} |
| 4 | **State & Data Integrity** | `PASS` / `WARN` / `FAIL` | {Summary of DB transactions, migrations, concurrency, or cache} |
| 5 | **UI / Rendering / UX** | `PASS` / `WARN` / `FAIL` | {Summary of component states, templates, styling, or error UX} |
| 6 | **Test Coverage & Quality** | `PASS` / `WARN` / `FAIL` | {Summary of unit/integration test coverage and assertion strength} |
| 7 | **Performance & Scalability** | `PASS` / `WARN` / `FAIL` | {Summary of N+1 queries, indexing, memory, or computational efficiency} |
| 8 | **Dead Code & Hygiene** | `PASS` / `WARN` / `FAIL` | {Summary of orphaned code, unused imports, or outdated comments} |
| 9 | **DRY & Architectural Consistency** | `PASS` / `WARN` / `FAIL` | {Summary of duplication, design pattern adherence, and naming} |

### Detailed Findings by Dimension

#### Dimension 1: Correctness
<!-- If clean: No issues identified. -->
| Severity | Location (`file:line`) | Description & Execution Path | Suggested Resolution |
|----------|------------------------|------------------------------|----------------------|
| `CRITICAL` / `MAJOR` / `MINOR` | `path/to/file.py:L45` | {Logic flaw explanation and runtime trace} | {Actionable fix} |

#### Dimension 2: Security & Auth
<!-- If clean: No issues identified. -->
| Severity | Location (`file:line`) | Vulnerability / Threat Scenario | Remediation |
|----------|------------------------|---------------------------------|-------------|
| `CRITICAL` / `MAJOR` / `MINOR` | `path/to/file.py:L88` | {Threat explanation, e.g. unauthenticated mutation} | {Required guard} |

#### Dimension 3: Regression Risk
<!-- If clean: No issues identified. -->
| Severity | Location (`file:line`) | Potential Broken Workflow | Verification Needed |
|----------|------------------------|---------------------------|---------------------|
| `MAJOR` / `MINOR` | `path/to/file.py:L12` | {Affected consumer or dependent module} | {Regression test case} |

#### Dimension 4: State & Data Integrity
<!-- If clean: No issues identified. -->
| Severity | Location (`file:line`) | Risk (Transaction / Migration / Concurrency) | Mitigation |
|----------|------------------------|----------------------------------------------|------------|
| `MAJOR` / `MINOR` | `path/to/models.py:L50` | {Schema or race condition description} | {Transaction wrapper / index} |

#### Dimension 5: UI / Rendering / UX
<!-- If clean: No issues identified. -->
| Severity | Location (`file:line`) | UI / Template / UX Flaw | Recommended Adjustment |
|----------|------------------------|-------------------------|------------------------|
| `MINOR` / `NIT` | `templates/booking.html:L20` | {Missing loading state or unescaped block} | {HTML / CSS / JS fix} |

#### Dimension 6: Test Coverage & Quality
<!-- If clean: No issues identified. -->
| Severity | Location (`file:line`) | Test Gap / Assertion Flaw | Test to Add |
|----------|------------------------|---------------------------|-------------|
| `MAJOR` / `MINOR` | `tests/test_service.py:L10` | {Untested edge condition} | {Specific test scenario} |

#### Dimension 7: Performance & Scalability
<!-- If clean: No issues identified. -->
| Severity | Location (`file:line`) | Performance Bottleneck (N+1 / Memory / Algorithmic) | Optimization |
|----------|------------------------|-----------------------------------------------------|--------------|
| `MAJOR` / `MINOR` | `path/to/views.py:L60` | {Iterative DB query in loop} | `select_related()` / batching |

#### Dimension 8: Dead Code & Hygiene
<!-- If clean: No issues identified. -->
| Severity | Location (`file:line`) | Unused Component / Orphaned Code | Action |
|----------|------------------------|----------------------------------|--------|
| `MINOR` / `NIT` | `path/to/utils.py:L15` | {Unused helper function or import} | Remove / deprecate |

#### Dimension 9: DRY & Architectural Consistency
<!-- If clean: No issues identified. -->
| Severity | Location (`file:line`) | Duplication / Architectural Drift | Refactoring Recommendation |
|----------|------------------------|-----------------------------------|----------------------------|
| `MINOR` / `NIT` | `path/to/views.py:L90` | {Duplicated logic found also in service.py:L30} | Extract to shared helper |

---

## 3. Key Findings & Inline Suggestions Index

### A. Blocking Findings (Must Resolve Prior to Merge)
1. **[CRITICAL]** `{Brief title of blocking issue 1}` (`path/to/file.py:L45`): {Summary and required resolution}
2. **[MAJOR]** `{Brief title of blocking issue 2}` (`path/to/file.py:L88`): {Summary and required resolution}

### B. Index of Placed Inline Suggestions
The following machine-actionable suggestions have been placed directly on the PR/MR diff:

| # | File | Line Range | Classification | Summary |
|---|------|------------|:--------------:|---------|
| 1 | `tutoring/services/slot_calculator.py` | L42-L45 | `[BUG]` | Handle `None` return from `get_active_term()` |
| 2 | `tutoring/views.py` | L88 | `[BUG]` | Replace naive UTC `timezone.now().date()` with `timezone.localdate()` |
| 3 | `tutoring/views.py` | L134-L138 | `[SECURITY]` | Enforce object-level parent ownership check on invoice retrieval |
| 4 | `tutoring/tasks/notifications.py` | L60-L65 | `[PERFORMANCE]` | Offload synchronous email dispatch to Celery background task |

### C. Recommended Next Steps for Author
1. [ ] Review and commit/apply inline code suggestions on the diff.
2. [ ] Address blocking findings in Section 3.A.
3. [ ] Run project test suite (`pytest` / `npm test`) to ensure zero regressions.
4. [ ] Push revised commit and re-request review.
```

---

## Verdict Determination Guidelines

| Verdict | Criteria | Provider Action |
|---------|----------|-----------------|
| `APPROVE` | - 100% Acceptance Criteria `VERIFIED`<br>- Zero `CRITICAL` or `MAJOR` findings<br>- All minor/nit suggestions are optional non-blockers | Submit review with `APPROVE` event (`gh pr review --approve` / `glab mr approve`). |
| `REQUEST_CHANGES` | - Any Acceptance Criteria `INCOMPLETE`, `MISSING`, or `DEVIATED`<br>- One or more `CRITICAL` or `MAJOR` issues in Correctness, Security, Regression, or Data Integrity | Submit review with `REQUEST_CHANGES` event (`gh pr review --request-changes`). |
| `COMMENT` | - Review provides informational analysis or clarification questions<br>- No blocking defects identified, but formal sign-off withheld pending discussion | Submit review with `COMMENT` event (`gh pr review --comment` / `glab mr note`). |
