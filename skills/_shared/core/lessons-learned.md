# Lessons Learned (Superseded & Retired)

> [!IMPORTANT]
> **This file is SUPERSEDED and RETIRED.**
> The monolithic `lessons-learned.md` structure has been replaced by the **2-Tier Knowledge Architecture**. Agents must not append lessons directly to this file. Instead, follow the Tier 1 / Tier 2 protocols detailed below.

---

## The 2-Tier Knowledge Architecture

Operational lessons and guardrails are decoupled into two distinct tiers to prevent context dilution, eliminate stale boilerplate, and enforce line-level verification rigour:

```
┌───────────────────────────────────────────────────────────────────────────┐
│ TIER 1: Active Operational Guardrails (docs/checklists/<domain>.md)       │
│ - Primary location: docs/checklists/<domain>.md (host project root)      │
│ - Pre-flight Tier 0 injection: Loaded before implementation (Phase 1/2)   │
│ - Format: 1-line actionable rules                                         │
│ - Verification: Enforced in Step 4 Verify with explicit file:line citations│
└───────────────────────────────────────────────────────────────────────────┘
                                      ▲
                                      │ Bug Closure Gate (Extract 1-line rule)
                                      │
┌───────────────────────────────────────────────────────────────────────────┐
│ TIER 2: Deep Incident Post-Mortems (.agents/results/bugs/)                │
│ - Primary location: .agents/results/bugs/                                 │
│ - Role: Full RCA artifacts for bugs, regressions, or high CD (>= 50)      │
│ - Format: Root cause, reproduction, fix, prevention, and test coverage    │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 3-State Graduation Lifecycle

Guardrails and operational lessons progress through three distinct lifecycle states:

1. **State 1: Incident (Post-Mortem)**
   - **Where**: `.agents/results/bugs/`
   - **When**: Created during bug investigation, regression analysis, or high Clarification Debt (CD >= 50).
   - **Content**: Detailed analysis of failure mechanics, reproduction steps, and root causes.

2. **State 2: Active Checklist (Operational Guardrail)**
   - **Where**: `docs/checklists/<domain>.md` (host project root, fallback: `.agents/skills/<skill>/resources/checklist.md`)
   - **When**: Extracted immediately upon bug resolution via the **Bug Closure Gate**.
   - **Content**: Lean, high-signal 1-line operational rules.
   - **Usage**: Injected pre-flight for domain agents; verified in Step 4 with explicit line citations (e.g. `- [x] Rule: Verified path/to/file.py:L42`).

3. **State 3: Graduated to Linter/Test (Automated Enforcement)**
   - **Where**: Linters (Ruff, ESLint), static analysis rules, CI checks, or regression test suites.
   - **When**: Codified into automated tooling.
   - **Lifecycle Action**: Once deterministic automation is in place, the rule is graduated out and removed from `docs/checklists/<domain>.md` to prevent checklist bloat.

---

## Domain Routing & Reference Pointers

Agents should reference the following resources instead of `lessons-learned.md`:

| Agent / Domain | Tier 1 Active Guardrails | Tier 2 Deep Post-Mortems |
|----------------|--------------------------|--------------------------|
| **Backend Agent (Impl)** | `docs/checklists/backend.md` (Tier 0 Domain Checklist) | `.agents/results/bugs/` |
| **Frontend Agent (Impl)** | `docs/checklists/frontend.md` (Tier 0 Domain Checklist) | `.agents/results/bugs/` |
| **Mobile Agent (Impl)** | `docs/checklists/mobile.md` (Tier 0 Domain Checklist) | `.agents/results/bugs/` |
| **Debug Agent** | Relevant `docs/checklists/<domain>.md` (Tier 0 Domain Checklist) | `.agents/results/bugs/` |
| **QA / Review Agent** | `docs/checklists/common.md` + `docs/checklists/<domain>.md` | `.agents/results/bugs/` |
| **PM / Plan Agent** | Macro Architectural Context (`ARCHITECTURE.md`, `docs/design-docs/`, `docs/product-specs/`, `docs/plans/work/tech-debt-tracker.md`) | `.agents/results/bugs/` |
| **Orchestrator** | `docs/checklists/<domain>.md` | `.agents/results/bugs/` |

---

## Protocol for Adding New Guardrails & RCAs

### 1. Bug Closure Gate (Debug / Implementation Agents)
When closing a bug in `.agents/results/bugs/`:
- Complete the bug report in `.agents/results/bugs/`.
- Extract a concise 1-line rule and append it to `docs/checklists/<domain>.md` using the standardized pattern:
  `- [ ] **{Topic}**: {Rule} (❌ Anti-pattern: `{X}`, ✅ Required: `{Y}`) [Ref: .agents/results/bugs/{incident-file}.md]`

### 2. High Clarification Debt (CD >= 50) & Review RCAs (QA Agent / Orchestrator)
When Clarification Debt threshold is breached (CD >= 50) or recurring review failures occur:
- Record the full RCA post-mortem in `.agents/results/bugs/`.
- Append the corresponding 1-line prevention rule to `docs/checklists/<domain>.md`.

### 3. Discarded Experiments (delta <= -5) (Orchestrator)
When an experiment is discarded with delta <= -5 in the Experiment Ledger:
- Save the post-mortem analysis in `.agents/results/bugs/`.
- Extract prevention guardrails into `docs/checklists/<domain>.md`.
