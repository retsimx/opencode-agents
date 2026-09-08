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
7. **Section 5: Out-of-Diff and Non-Blocking Observations**: Valid findings on untouched code outside diff hunks (and non-blocking in-hunk observations) to guarantee zero HTTP 422 API errors; never published as inline comments.
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
> - `OPERATIONAL` (or `PENDING_EXTERNAL_EVIDENCE`): Requirement cannot be verified from the MR alone (e.g. requires a live deployment, external service, or runtime gate). This is **not** a code defect and does not by itself force `REQUEST_CHANGES`; it may produce `COMMENT` when the evidence is a required pre-merge gate, otherwise it is a deployment follow-up.
> - `AMBIGUOUS`: Normative criteria that conflict or cannot be resolved from current evidence. Do **not** invent a precedence rule; preserve concrete findings and use `COMMENT` unless an independently verified merge-safety defect requires `REQUEST_CHANGES`.

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
- **Disposition**: `BLOCKING` / `NON-BLOCKING`
- **Contract Basis**: `Criterion #N` / `EXTRA-CONTRACT MERGE-SAFETY` / `ADVISORY`
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
| Severity | Disposition | Contract Basis | Location (`file:line`) | Description & Execution Path | Suggested Resolution |
|----------|:-----------:|:--------------:|------------------------|------------------------------|----------------------|
| `CRITICAL` / `MAJOR` / `MINOR` | `BLOCKING` / `NON-BLOCKING` | `Criterion #N` / `EXTRA-CONTRACT MERGE-SAFETY` / `ADVISORY` | `path/to/file.py:L45` | {Logic flaw explanation and runtime trace} | {Actionable fix} |

#### Dimension 2: Security & Auth
<!-- If clean: No security/auth issues identified. -->
| Severity | Disposition | Contract Basis | Location (`file:line`) | Vulnerability / Auth Gap | Remediation |
|----------|:-----------:|:--------------:|------------------------|--------------------------|-------------|
| `CRITICAL` / `MAJOR` / `MINOR` | `BLOCKING` / `NON-BLOCKING` | `Criterion #N` / `EXTRA-CONTRACT MERGE-SAFETY` / `ADVISORY` | `path/to/file.py:L88` | {Auth gap explanation} | {Required permission guard} |

#### Dimension 3: Regression Risk
<!-- If clean: No regression risks identified. -->
| Severity | Disposition | Contract Basis | Location (`file:line`) | Potential Broken Workflow | Verification Needed |
|----------|:-----------:|:--------------:|------------------------|---------------------------|---------------------|
| `MAJOR` / `MINOR` | `BLOCKING` / `NON-BLOCKING` | `Criterion #N` / `EXTRA-CONTRACT MERGE-SAFETY` / `ADVISORY` | `path/to/file.py:L12` | {Affected consumer or dependent module} | {Regression test case} |

#### Dimension 4: State & Data Integrity
<!-- If clean: No state/data integrity risks identified. -->
| Severity | Disposition | Contract Basis | Location (`file:line`) | Risk (Transaction / Migration / Concurrency) | Mitigation |
|----------|:-----------:|:--------------:|------------------------|----------------------------------------------|------------|
| `MAJOR` / `MINOR` | `BLOCKING` / `NON-BLOCKING` | `Criterion #N` / `EXTRA-CONTRACT MERGE-SAFETY` / `ADVISORY` | `path/to/models.py:L50` | {Schema or race condition description} | {Transaction wrapper / index} |

#### Dimension 5: UI / Rendering / UX
<!-- If clean: No UI/UX flaws identified. -->
| Severity | Disposition | Contract Basis | Location (`file:line`) | UI / Template / UX Flaw | Recommended Adjustment |
|----------|:-----------:|:--------------:|------------------------|-------------------------|------------------------|
| `MINOR` / `NIT` | `BLOCKING` / `NON-BLOCKING` | `Criterion #N` / `EXTRA-CONTRACT MERGE-SAFETY` / `ADVISORY` | `templates/booking.html:L20` | {Missing loading state or unescaped block} | {HTML / CSS / JS fix} |

#### Dimension 6: Test Coverage & Quality
<!-- If clean: Test coverage meets standards. -->
| Severity | Disposition | Contract Basis | Location (`file:line`) | Test Gap / Assertion Flaw | Test to Add |
|----------|:-----------:|:--------------:|------------------------|---------------------------|-------------|
| `MAJOR` / `MINOR` | `BLOCKING` / `NON-BLOCKING` | `Criterion #N` / `EXTRA-CONTRACT MERGE-SAFETY` / `ADVISORY` | `tests/test_service.py:L10` | {Untested edge condition} | {Specific test scenario} |

#### Dimension 7: Performance & Scalability
<!-- If clean: No performance bottlenecks identified. -->
| Severity | Disposition | Contract Basis | Location (`file:line`) | Performance Bottleneck (N+1 / Memory / Algorithmic) | Optimization |
|----------|:-----------:|:--------------:|------------------------|-----------------------------------------------------|--------------|
| `MAJOR` / `MINOR` | `BLOCKING` / `NON-BLOCKING` | `Criterion #N` / `EXTRA-CONTRACT MERGE-SAFETY` / `ADVISORY` | `path/to/views.py:L60` | {Iterative DB query in loop} | `select_related()` / batching |

#### Dimension 8: Dead Code & Hygiene
<!-- If clean: Codebase is clean and hygienic. -->
| Severity | Disposition | Contract Basis | Location (`file:line`) | Unused Component / Orphaned Code | Action |
|----------|:-----------:|:--------------:|------------------------|----------------------------------|--------|
| `MINOR` / `NIT` | `BLOCKING` / `NON-BLOCKING` | `Criterion #N` / `EXTRA-CONTRACT MERGE-SAFETY` / `ADVISORY` | `path/to/utils.py:L15` | {Unused helper function or import} | Remove / deprecate |

#### Dimension 9: DRY & Architectural Consistency
<!-- If clean: Architecture and DRY standards maintained. -->
| Severity | Disposition | Contract Basis | Location (`file:line`) | Duplication / Architectural Drift | Refactoring Recommendation |
|----------|:-----------:|:--------------:|------------------------|-----------------------------------|----------------------------|
| `MINOR` / `NIT` | `BLOCKING` / `NON-BLOCKING` | `Criterion #N` / `EXTRA-CONTRACT MERGE-SAFETY` / `ADVISORY` | `path/to/views.py:L90` | {Duplicated logic found also in service.py:L30} | Extract to shared helper |

### Grug Compliance (Cross-Cutting Lens)

> **Cross-cutting lens, NOT a tenth dimension.** This subsection applies the Grug principles as an orthogonal complexity lens over the 9 dimensions above. It does not add a 10th dimension and does not change the "9-Dimension" scope of Section 3.

**Aggregate Grug Verdict**: `{PASS | WARN | FAIL}`

Report against the **Minimum Grug Reporting Shortlist** in `.agents/rules/grug-principles.md` (rule numbers are GLOBAL 1–28). Only rules with review-applicable findings are listed; rules with no findings are omitted.

| Rule # | Status | Finding | Location (`file:line`) |
|--------|:------:|---------|------------------------|
| `{2}` | `PASS` / `WARN` / `FAIL` | {Complexity / minimalism finding} | `path/to/file.py:L45` |
| `{3}` | `PASS` / `WARN` / `FAIL` | {Premature factoring finding} | `path/to/file.py:L88` |
| `{4}` | `PASS` / `WARN` / `FAIL` | {Simple-over-clever finding} | `path/to/file.py:L12` |

> **INFORMATIONAL**: The aggregate Grug verdict is informational only and does **not** independently determine approval. However, Grug rules that prohibit invented requirements and unnecessary machinery **constrain** finding severity, blocking disposition, and remediation scope. A finding cannot block solely on speculative complexity, and its requested fix must be the smallest change that resolves the demonstrated contract deviation or reachable defect. Actionable Grug violations flow through the existing severity levels (CRITICAL / MAJOR / MINOR) and the `BLOCKING` / `NON-BLOCKING` disposition already reported in the 9-dimension detailed findings and Section 4 staged suggestions. A `FAIL` Grug verdict alone does not force `REQUEST_CHANGES`; it only signals complexity concerns to weigh alongside the dimension-level severities and dispositions.

---

## 4. Staged Inline Diff Suggestions & Detailed Remediation (Subagent 4 Verified)

The following actionable findings fall strictly within modified diff hunks and are verified for line coordinates, syntax validity, and indentation:

### 1. [{SEVERITY}] [{CLASSIFICATION}]: {Short Title}
- **Location**: `{file:line}` (in modified diff hunk)
- **Disposition**: `BLOCKING` / `NON-BLOCKING`
- **Contract Basis**: `Criterion #N` / `EXTRA-CONTRACT MERGE-SAFETY` / `ADVISORY`
- **Problem**: {Detailed failure mode and runtime consequence}
- **Remediation**: {Concrete fix description}

```suggestion
{exact replacement code matching source indentation}
```

### 2. [{SEVERITY}] [{CLASSIFICATION}]: {Short Title}
- **Location**: `{file:start_line-end_line}` (in modified diff hunk)
- **Disposition**: `BLOCKING` / `NON-BLOCKING`
- **Contract Basis**: `Criterion #N` / `EXTRA-CONTRACT MERGE-SAFETY` / `ADVISORY`
- **Problem**: {Detailed failure mode and runtime consequence}
- **Remediation**: {Concrete fix description}

```suggestion
{exact replacement code matching source indentation}
```

---

## 5. Out-of-Diff and Non-Blocking Observations

> **Placement routing (independent of disposition)**: Section placement does **not** determine disposition — disposition determines the verdict, not placement or technical severity.
> - `BLOCKING` + in-hunk → Section 4 (eligible for inline publication).
> - `NON-BLOCKING` + in-hunk → Section 4 only when a concise local suggestion is useful; otherwise Section 5.
> - Any out-of-hunk finding → Section 5.
> - Section 5 findings are **never** published as inline comments.

The following valid findings target unmodified lines outside active diff hunks (or are non-blocking observations). They have been demoted from inline comments to top-level review observations to guarantee zero Forge API HTTP 422 errors while providing complete engineering feedback:

| # | Severity | Disposition | Contract Basis | Classification | Location (`file:line`) | Observation / Defect | Recommended Resolution |
|---|:--------:|:-----------:|:--------------:|:--------------:|------------------------|----------------------|------------------------|
| 1 | `MAJOR` | `NON-BLOCKING` | `ADVISORY` | `[SECURITY]` | `path/to/legacy_view.py:L42` | Pre-existing missing permission check on caller | Wrap caller with permission decorator |
| 2 | `MINOR` | `NON-BLOCKING` | `ADVISORY` | `[PERFORMANCE]` | `path/to/helpers.py:L115` | Pre-existing unindexed queryset lookup | Add database index on lookup column |

---

## 5.5 Previously Raised, Now Verified Fixed (Optional — Verifier Courtesy)

> **OPTIONAL**: This subsection is a courtesy note produced only when the verifier cross-references prior-round findings (`pr-history.md`). It is not required and does not affect the verdict. Each entry must cite a current-head `file:line` proving the prior finding is now resolved.

| # | Previously Raised (Source Round) | Current Location (`file:line`) | Verification Detail |
|---|----------------------------------|-------------------------------|---------------------|
| 1 | `{prior finding summary} — round {N}` | `path/to/file.py:L88` | {How the current head resolves the prior finding, verified at this location} |

---

## 6. Recommended Next Steps for Author

- [ ] **Apply Inline Suggestions**: Review and accept/commit the verified inline suggestion blocks on the diff.
- [ ] **Address Blocking Findings**: Address every finding marked `BLOCKING` in Sections 2, 3, and 4. Treat `NON-BLOCKING` findings as optional follow-up unless the contract is clarified to require them.
- [ ] **Fulfill Missing Criteria**: Complete any `INCOMPLETE`, `DEVIATED`, or `MISSING` Acceptance Criteria identified in Section 1.
- [ ] **Out-of-Diff Follow-ups**: Create tracking issues for out-of-diff observations listed in Section 5 if outside PR scope.
- [ ] **Run Test Suite**: Execute full local test suite (`pytest` / `npm test`) to confirm zero regressions.
- [ ] **Push & Re-request Review**: Push updated commits and re-request review on the forge.
```

---

## Verdict Determination Guidelines

The verdict is **contract-led**, determined in the following order:

1. **Contract baseline**: `REQUEST_CHANGES` if any explicit normative acceptance criterion is `INCOMPLETE`, `DEVIATED`, or `MISSING`.
2. **Merge-safety override**: `REQUEST_CHANGES` only for a **verified** `BLOCKING` finding proving a concrete reachable correctness regression, data-loss / integrity failure, or `CRITICAL` / `HIGH` security vulnerability — even when omitted from the contract.
3. **Non-blocking findings remain visible** and do not force `REQUEST_CHANGES`, regardless of hypothetical impact severity.

`OPERATIONAL` / `PENDING_EXTERNAL_EVIDENCE` and `AMBIGUOUS` criterion statuses are not code defects and do not by themselves force `REQUEST_CHANGES` (see Section 1 legend). Section placement and technical severity do not determine the verdict — disposition does.

| Verdict | Criteria | Provider Action |
|---------|----------|-----------------|
| `APPROVE` | - No criterion is `INCOMPLETE`, `DEVIATED`, or `MISSING`<br>- Zero `BLOCKING` findings across Quality and Security audits<br>- All required pre-merge gates verified<br>- Non-gating `OPERATIONAL` follow-ups may remain<br>- All non-blocking findings are optional follow-up | Submit review with `APPROVE` event (`gh pr review --approve` / `glab mr approve`). |
| `REQUEST_CHANGES` | - Any Acceptance Criteria `INCOMPLETE`, `MISSING`, or `DEVIATED` (contract baseline)<br>- A verified `BLOCKING` merge-safety defect: concrete reachable correctness regression, data-loss / integrity failure, or `CRITICAL` / `HIGH` security vulnerability (merge-safety override)<br>- Unresolved high-risk security vulnerabilities that meet the verified reachable bar | Submit review with `REQUEST_CHANGES` event (`gh pr review --request-changes` / `glab mr unapprove`). |
| `COMMENT` | - Review provides informational analysis, clarification questions, or architectural feedback<br>- No `BLOCKING` findings identified, but formal sign-off withheld pending author discussion<br>- `OPERATIONAL` / `AMBIGUOUS` criteria present where evidence is a required pre-merge gate | Submit review with `COMMENT` event (`gh pr review --comment` / `glab mr note`). |
