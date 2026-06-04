---
name: deepinit
description: >
  Initialize or reinitialize a project's AI harness — AGENTS.md table of
  contents, ARCHITECTURE.md domain map, and structured docs/ knowledge base
  with design docs, plans, product specs, and domain guides. Use when setting
  up a new project for AI-assisted development or regenerating an existing
  project's documentation harness from scratch.
---

# Project Initialization (`deepinit`)

## Scheduling

### Goal
Bootstrap a project's AI-readable knowledge base: AGENTS.md as a ~100-line table of contents, ARCHITECTURE.md as the domain-layering map, and a structured `docs/` tree covering design decisions, plans, product specs, and domain guides.

### Intent signature
- User runs `/deepinit` or says "initialize project", "set up docs", "create AGENTS.md"
- User wants to bootstrap AI-assisted development harness for a new or existing project
- Project lacks AGENTS.md, ARCHITECTURE.md, or structured documentation

### When to use
Run this skill when the project has no AGENTS.md, or the existing harness is stale and needs regeneration. Runs inline (no subagent spawning).

## Structural Flow

Execute `.agents/workflows/deepinit.md` which covers the full bootstrap sequence: codebase analysis, ARCHITECTURE.md generation, docs/ tree population, AGENTS.md table of contents, boundary AGENTS.md files, and validation.

## Logical Operations

All logic is defined in the workflow file at `.agents/workflows/deepinit.md`. Load and follow it step-by-step without deviation.

## References

- Workflow: `.agents/workflows/deepinit.md`
- Coordination skill: `.agents/skills/coordination/SKILL.md`
- Context loading: `.agents/skills/_shared/core/context-loading.md`
