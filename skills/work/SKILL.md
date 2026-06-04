---
name: work
description: >
  Coordinate multiple agents for a complex multi-domain project using PM
  planning, parallel agent spawning, and QA review. Use for multi-domain
  features needing backend + frontend + mobile + QA, or any task that
  requires PM decomposition followed by parallel implementation and review.
---

# Multi-Agent Work Coordination (`work`)

## Scheduling

### Goal
Orchestrate multi-domain feature development: PM-driven task breakdown, parallel backend/frontend/mobile implementation, QA review, and iterative fix-verify loops. Single-domain tasks should use the domain agent directly instead.

### Intent signature
- User says "build this feature", "implement across backend and frontend", "work on this"
- Task spans multiple domains (backend + frontend + mobile + QA)
- User wants PM planning followed by structured execution
- User explicitly invokes `/work`

### When to use
For multi-domain features requiring coordination across backend, frontend, mobile, and/or QA. Single-domain tasks should be routed to the domain skill directly. Always get user confirmation on the plan before spawning agents.

## Structural Flow

Execute `.agents/workflows/work.md` which covers the full 8-step sequence: preparation, requirements analysis, PM agent task decomposition, plan review with user, priority-tiered agent spawning, progress monitoring, QA review, and fix-verify iteration.

## Logical Operations

All logic is defined in the workflow file at `.agents/workflows/work.md`. Load and follow it step-by-step without deviation.

## References

- Workflow: `.agents/workflows/work.md`
- Coordination skill: `.agents/skills/coordination/SKILL.md`
- Context loading: `.agents/skills/_shared/core/context-loading.md`
- Memory protocol: `.agents/skills/_shared/runtime/memory-protocol.md`
