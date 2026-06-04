---
name: stack-set
description: >
  Auto-detect a project's tech stack by scanning manifests
  (Cargo.toml, package.json, pyproject.toml, go.mod, etc.) and generate
  stack-specific language/framework references — stack.yaml, tech-stack.md,
  snippets.md, and API boilerplate — into the backend skill's stack/
  directory. Use when setting up or updating a project's tech stack
  configuration for AI tooling.
---

# Stack Configuration (`stack-set`)

## Scheduling

### Goal
Detect the project's language, framework, ORM, validation, migration, and test tools from manifests, then generate stack-specific reference files for consistent AI-generated code.

### Intent signature
- User says "set up stack", "detect tech stack", "configure stack"
- Project needs stack-specific code patterns generated
- Backend skill's stack/ directory is missing or out of date

### When to use
On initial project setup, after adding new dependencies, or when stack-specific code generation produces incorrect patterns.

## Structural Flow

Execute `.agents/workflows/stack-set.md` which covers the full 4-step sequence: manifest detection, confirmation, file generation (stack.yaml, tech-stack.md, snippets.md, api-template), and verification.

## Logical Operations

All logic is defined in the workflow file at `.agents/workflows/stack-set.md`. Load and follow it step-by-step without deviation.

## References

- Workflow: `.agents/workflows/stack-set.md`
- Backend skill: `.agents/skills/backend/SKILL.md`
