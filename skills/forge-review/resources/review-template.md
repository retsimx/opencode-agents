# Forge Review: Top-Level PR/MR Review Summary Template

This resource defines the authoritative, standardized 6-Section Review Deliverable Schema for top-level review summaries posted to GitHub Pull Requests (`gh`) or GitLab Merge Requests (`glab`) by `forge-review`.

---

## 6-Section Review Deliverable Schema Specification

The deliverable follows a strict 6-section hierarchy designed for maximum audit rigor, contract alignment, zero-trust security isolation, and machine-actionable inline diff remediation:

1. **Header & Verdict**: PR metadata, evaluated commit range, target branch, linked issue/epic, and top-level verdict.
2. **Executive Summary**: 2-4 sentence executive overview.
3. **Section 1: Acceptance Criteria & Contract Alignment Matrix**: Issue/Epic requirement verification with `file:line` proof citations.
4. **Section 2: Dedicated Security & Threat Model Audit (Subagent 3 Zero-Trust Pass)**: Zero-trust audit across 6 threat vectors with concrete exploit scenarios.
5. **Section 3: 9-Dimension Code Quality & Architecture Audit Scorecard (Subagent 2 Deep Review)**: Comprehensive 9-dimension quality scorecard and breakdown.
6. **Section 4: Staged Inline Diff Suggestions & Detailed Remediation (Subagent 4 Verified)**: Verified inline findings formatted with 4-part breakdown and ```` ```suggestion ```` blocks.
7. **Section 5: Out-of-Diff Observations (Demoted from Inline)**: Valid findings on untouched code outside diff hunks to guarantee zero HTTP 422 API errors.
8. **Section 6: Recommended Next Steps for Author**: Actionable checklist for the PR author.

---

## Top-Level Review Markdown Template

```markdown
# Code Review Summary: PR #{PR_NUMBER} — {PR_TITLE}

**Review Verdict**: `{VERDICT}` <!-- Options: APPROVE | REQUEST_CHANGES | COMMENT -->
**Evaluated Scope**: `{COMMIT_RANGE}` (`{HEAD_SHA}`)
**Target Branch**: `{BASE_BRANCH}`
**Associated Issue / Epic**: Issue #{ISSUE_NUMBER} (Epic #{EPIC_NUMBER})
**Review Date**: `{DATE_TIME_UTC}`

---

## Executive Summary

{2-4 sentence executive overview summarizing the scope of changes, overall code quality, acceptance criteria compliance, critical security findings, and primary actions required from the author before merge.}

---

## 1. Acceptance Criteria & Contract Alignment Matrix

This matrix verifies that 100% of requirements from the associated Issue and Epic specifications are fully implemented, backed by concrete code citations in the workspace.

| # | Requirement / Acceptance Criterion | Source Ref | Status | Code Proof (`file:line`) | Notes / Verification Details |
|---|-------------------------------------|------------|:------:|--------------------------|------------------------------|
| 1 | {Requirement description 1} | Issue #{NUM} / Epic #{NUM} | `VERIFIED` | `path/to/file.py:L45-L52` | {Implementation details and test verification note} |
| 2 | {Requirement description 2} | Issue #{NUM} | `VERIFIED` | `path/to/view.py:L102` | {Tested in tests/test_suite.py:L33} |
| 3 | {Requirement description 3} | Issue #{NUM} | `INCOMPLETE` / `DEVIATED` / `MISSING` | `path/to/handler.py:L80` | {Specific missing edge case or contract deviation} |

> **Status Legend**:
> - `VERIFIED`: Requirement is completely implemented, covered by tests, and adheres strictly to contract.
> - `INCOMPLETE`: Partially implemented; secondary edge cases, error handlers, or parameters missing.
> - `DEVIATED`: Implemented differently than specified in the issue/epic contract without documented rationale.
> - `MISSING`: Requirement was specified in the issue/epic but has no implementation in the diff.

---

## 2. Dedicated Security & Threat Model Audit (Subagent 3 Zero-Trust Pass)

**Security Verdict**: `{CLEAN | VULNERABILITY DETECTED}`

Evaluated under strict Zero-Trust isolation (diff only, no author narrative assumptions) across the 6 core threat vectors:

| # | Threat Vector | Status | Findings / Risk Evaluation |
|---|---------------|:------:|----------------------------|
| 1 | **Authentication & Sessions** | `CLEAN` / `VULNERABLE` | {Token handling, session fixation, credential storage, auth guard coverage} |
| 2 | **Authorization & IDOR / Tenancy** | `CLEAN` / `VULNERABLE` | {Object-level permissions, tenant scoping, privilege escalation, unvalidated ownership} |
| 3 | **Injection Flaws (SQLi / XSS / Command)** | `CLEAN` / `VULNERABLE` | {Unsanitized raw SQL, unescaped HTML/template rendering, unsafe subprocess invocation} |
| 4 | **CSRF & State Mutation Protection** | `CLEAN` / `VULNERABLE` | {Missing CSRF tokens, unsafe GET state mutations, permissive CORS policies} |
| 5 | **Sensitive Data / PII Exposure & Logging** | `CLEAN` / `VULNERABLE` | {Hardcoded credentials, unmasked PII, sensitive data in logs or client payloads} |
| 6 | **Cryptography & Insecure Dependencies** | `CLEAN` / `VULNERABLE` | {Weak hashing/ciphers, outdated vulnerable dependencies, insecure random generators} |

### Concrete Exploit Scenarios & Security Remediation

<!-- If Clean: No security vulnerabilities identified during zero-trust pass. -->

#### Threat Vector: {Vector Name (e.g., Authorization & IDOR)}
- **Location**: `{path/to/file.ext}:{line_or_range}`
- **Exploit Path & Impact**: {Detailed step-by-step description of how an attacker could exploit this flaw, prerequisites, payload, and direct business/technical impact.}
- **Remediation**:
```suggestion
{exact remediation code matching indentation}
```

---

## 3. 9-Dimension Code Quality & Architecture Audit Scorecard (Subagent 2 Deep Review)

Comprehensive evaluation across the 9 core software engineering dimensions:

| # | Dimension | Status | Summary & Risk Assessment |
|---|-----------|:------:|---------------------------|
| 1 | **Correctness** | `PASS` / `WARN` / `FAIL` | {Logic errors, unhandled boundary conditions, broken invariants} |
| 2 | **Security & Auth** | `PASS` / `WARN` / `FAIL` | {Baseline code-level auth checks, input sanitization, permission guards} |
| 3 | **Regression Risk** | `PASS` / `WARN` / `FAIL` | {Backward compatibility, broken existing consumers, changed contracts} |
| 4 | **State & Data Integrity** | `PASS` / `WARN` / `FAIL` | {DB transactions, schema migrations, race conditions, cache consistency} |
| 5 | **UI / Rendering / UX** | `PASS` / `WARN` / `FAIL` | {Component states, template escaping, responsive layout, error feedback} |
| 6 | **Test Coverage & Quality** | `PASS` / `WARN` / `FAIL` | {Unit/integration test coverage, edge cases, deterministic assertions} |
| 7 | **Performance & Scalability** | `PASS` / `WARN` / `FAIL` | {N+1 queries, memory bottlenecks, algorithmic complexity, indexing} |
| 8 | **Dead Code & Hygiene** | `PASS` / `WARN` / `FAIL` | {Orphaned code, unused imports, obsolete comments, lint warnings} |
| 9 | **DRY & Architectural Consistency** | `PASS` / `WARN` / `FAIL` | {Code duplication, modularity, pattern adherence, idiomatic conventions} |

### Detailed Findings by Dimension

#### Dimension 1: Correctness
<!-- If clean: No correctness issues identified. -->
| Severity | Location (`file:line`) | Description & Execution Path | Suggested Resolution |
|----------|------------------------|------------------------------|----------------------|
| `CRITICAL` / `MAJOR` / `MINOR` | `path/to/file.py:L45` | {Logic flaw explanation and runtime trace} | {Actionable fix} |

#### Dimension 2: Security & Auth
<!-- If clean: No security/auth issues identified. -->
| Severity | Location (`file:line`) | Vulnerability / Auth Gap | Remediation |
|----------|------------------------|--------------------------|-------------|
| `CRITICAL` / `MAJOR` / `MINOR` | `path/to/file.py:L88` | {Auth gap explanation} | {Required permission guard} |

#### Dimension 3: Regression Risk
<!-- If clean: No regression risks identified. -->
| Severity | Location (`file:line`) | Potential Broken Workflow | Verification Needed |
|----------|------------------------|---------------------------|---------------------|
| `MAJOR` / `MINOR` | `path/to/file.py:L12` | {Affected consumer or dependent module} | {Regression test case} |

#### Dimension 4: State & Data Integrity
<!-- If clean: No state/data integrity risks identified. -->
| Severity | Location (`file:line`) | Risk (Transaction / Migration / Concurrency) | Mitigation |
|----------|------------------------|----------------------------------------------|------------|
| `MAJOR` / `MINOR` | `path/to/models.py:L50` | {Schema or race condition description} | {Transaction wrapper / index} |

#### Dimension 5: UI / Rendering / UX
<!-- If clean: No UI/UX flaws identified. -->
| Severity | Location (`file:line`) | UI / Template / UX Flaw | Recommended Adjustment |
|----------|------------------------|-------------------------|------------------------|
| `MINOR` / `NIT` | `templates/booking.html:L20` | {Missing loading state or unescaped block} | {HTML / CSS / JS fix} |

#### Dimension 6: Test Coverage & Quality
<!-- If clean: Test coverage meets standards. -->
| Severity | Location (`file:line`) | Test Gap / Assertion Flaw | Test to Add |
|----------|------------------------|---------------------------|-------------|
| `MAJOR` / `MINOR` | `tests/test_service.py:L10` | {Untested edge condition} | {Specific test scenario} |

#### Dimension 7: Performance & Scalability
<!-- If clean: No performance bottlenecks identified. -->
| Severity | Location (`file:line`) | Performance Bottleneck (N+1 / Memory / Algorithmic) | Optimization |
|----------|------------------------|-----------------------------------------------------|--------------|
| `MAJOR` / `MINOR` | `path/to/views.py:L60` | {Iterative DB query in loop} | `select_related()` / batching |

#### Dimension 8: Dead Code & Hygiene
<!-- If clean: Codebase is clean and hygienic. -->
| Severity | Location (`file:line`) | Unused Component / Orphaned Code | Action |
|----------|------------------------|----------------------------------|--------|
| `MINOR` / `NIT` | `path/to/utils.py:L15` | {Unused helper function or import} | Remove / deprecate |

#### Dimension 9: DRY & Architectural Consistency
<!-- If clean: Architecture and DRY standards maintained. -->
| Severity | Location (`file:line`) | Duplication / Architectural Drift | Refactoring Recommendation |
|----------|------------------------|-----------------------------------|----------------------------|
| `MINOR` / `NIT` | `path/to/views.py:L90` | {Duplicated logic found also in service.py:L30} | Extract to shared helper |

---

## 4. Staged Inline Diff Suggestions & Detailed Remediation (Subagent 4 Verified)

The following actionable findings fall strictly within modified diff hunks and are verified for line coordinates, syntax validity, and indentation:

### 1. [{SEVERITY}] [{CLASSIFICATION}]: {Short Title}
- **Location**: `{file:line}` (in modified diff hunk)
- **Problem**: {Detailed failure mode and runtime consequence}
- **Remediation**: {Concrete fix description}

```suggestion
{exact replacement code matching source indentation}
```

### 2. [{SEVERITY}] [{CLASSIFICATION}]: {Short Title}
- **Location**: `{file:start_line-end_line}` (in modified diff hunk)
- **Problem**: {Detailed failure mode and runtime consequence}
- **Remediation**: {Concrete fix description}

```suggestion
{exact replacement code matching source indentation}
```

---

## 5. Out-of-Diff Observations (Demoted from Inline)

The following valid findings target unmodified lines outside active diff hunks. They have been demoted from inline comments to top-level review observations to guarantee zero Forge API HTTP 422 errors while providing complete engineering feedback:

| # | Severity | Classification | Location (`file:line`) | Observation / Defect | Recommended Resolution |
|---|:--------:|:--------------:|------------------------|----------------------|------------------------|
| 1 | `MAJOR` | `[SECURITY]` | `path/to/legacy_view.py:L42` | Pre-existing missing permission check on caller | Wrap caller with permission decorator |
| 2 | `MINOR` | `[PERFORMANCE]` | `path/to/helpers.py:L115` | Pre-existing unindexed queryset lookup | Add database index on lookup column |

---

## 5.5 Previously Raised, Now Verified Fixed (Optional — Verifier Courtesy)

> **OPTIONAL**: This subsection is a courtesy note produced only when the verifier cross-references prior-round findings (`pr-history.md`). It is not required and does not affect the verdict. Each entry must cite a current-head `file:line` proving the prior finding is now resolved.

| # | Previously Raised (Source Round) | Current Location (`file:line`) | Verification Detail |
|---|----------------------------------|-------------------------------|---------------------|
| 1 | `{prior finding summary} — round {N}` | `path/to/file.py:L88` | {How the current head resolves the prior finding, verified at this location} |

---

## 6. Recommended Next Steps for Author

- [ ] **Apply Inline Suggestions**: Review and accept/commit the verified inline suggestion blocks on the diff.
- [ ] **Address Blocking Findings**: Resolve all `CRITICAL` and `MAJOR` issues in Sections 2, 3, and 4.
- [ ] **Fulfill Missing Criteria**: Complete any `INCOMPLETE`, `DEVIATED`, or `MISSING` Acceptance Criteria identified in Section 1.
- [ ] **Out-of-Diff Follow-ups**: Create tracking issues for out-of-diff observations listed in Section 5 if outside PR scope.
- [ ] **Run Test Suite**: Execute full local test suite (`pytest` / `npm test`) to confirm zero regressions.
- [ ] **Push & Re-request Review**: Push updated commits and re-request review on the forge.
```

---

## Verdict Determination Guidelines

| Verdict | Criteria | Provider Action |
|---------|----------|-----------------|
| `APPROVE` | - 100% Acceptance Criteria `VERIFIED`<br>- Zero `CRITICAL` or `MAJOR` findings across Quality and Security audits<br>- All minor/nit suggestions are optional non-blockers | Submit review with `APPROVE` event (`gh pr review --approve` / `glab mr approve`). |
| `REQUEST_CHANGES` | - Any Acceptance Criteria `INCOMPLETE`, `MISSING`, or `DEVIATED`<br>- One or more `CRITICAL` or `MAJOR` issues in Correctness, Security, Regression, or Data Integrity<br>- Unresolved high-risk security vulnerabilities | Submit review with `REQUEST_CHANGES` event (`gh pr review --request-changes` / `glab mr unapprove`). |
| `COMMENT` | - Review provides informational analysis, clarification questions, or architectural feedback<br>- No blocking defects identified, but formal sign-off withheld pending author discussion | Submit review with `COMMENT` event (`gh pr review --comment` / `glab mr note`). |
