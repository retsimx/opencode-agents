---
name: ultrawork
description: >
  High-quality 5-phase development workflow with 11 review steps out of 17
  total — PLAN (PM-driven task breakdown), IMPL (parallel agent spawning),
  VERIFY (QA alignment/safety/regression), REFINE (deep review, split large
  files, side-effect analysis), and SHIP (final QA, UX, deployment readiness).
  Use for high-stakes features requiring deep quality gates at every phase.
---

# Ultrawork — High-Quality 5-Phase Development (`ultrawork`)

## Scheduling

### Goal
Drive complex, multi-phase development with deep quality gates at each of 5 phases: PLAN, IMPL, VERIFY, REFINE, SHIP. Includes 11 review steps, conditional Quality Score measurement, Experiment Ledger tracking, and Exploration Loop for stubborn issues.

### Intent signature
- User says "ultrawork", "high quality", "5-phase", "deep review"
- Task has high business impact or risk
- User wants thorough QA gates at every phase, not just at the end
- User explicitly invokes `/ultrawork`

### When to use
For high-stakes or complex features where a standard work cycle is insufficient. Requires subagent spawning (PM, implementation agents, QA, debug). Always get user confirmation at the PLAN_GATE before proceeding to implementation.

## Structural Flow

Execute `.agents/workflows/ultrawork.md` which covers all 5 phases with their gates, subagent delegation patterns, Quality Score measurement, Experiment Ledger, and the Exploration Loop for repeated gate failures.

## Logical Operations

All logic is defined in the workflow file at `.agents/workflows/ultrawork.md`. Load and follow it step-by-step without deviation.

## References

- Workflow: `.agents/workflows/ultrawork.md`
- Coordination skill: `.agents/skills/coordination/SKILL.md`
- Quality principles: `.agents/skills/_shared/core/quality-principles.md`
- Context loading: `.agents/skills/_shared/core/context-loading.md`
- Memory protocol: `.agents/skills/_shared/runtime/memory-protocol.md`
- Exploration loop: `.agents/skills/_shared/conditional/exploration-loop.md`
- Phase gates: `.agents/workflows/ultrawork/resources/phase-gates.md`
- Multi-review protocol: `.agents/workflows/ultrawork/resources/multi-review-protocol.md`
