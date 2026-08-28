---
name: stack-set
description: Auto-detect a project's tech stack by scanning manifests (Cargo.toml, package.json, pyproject.toml, go.mod, etc.) and generate stack-specific language/framework references — stack.yaml, tech-stack.md, snippets.md, and API boilerplate — into the backend skill's stack/ directory, and seed host project domain checklists in docs/checklists/. Use when setting up or updating a project's tech stack configuration for AI tooling.
---

# Stack Configuration

## Goal
Analyze project files to detect the tech stack, generate language-specific references in the domain skill's `stack/` directory, and seed/update host project domain checklists in `docs/checklists/`.

> **Vendor note:** This workflow executes inline (no subagent spawning). All vendors use their native file reading tools for manifest detection and file writing tools for stack generation.

---

## Step 1: Detect

Scan project root for package manifests:

| File | Detection |
|:---|:---|
| `pyproject.toml`, `requirements.txt`, `Pipfile` | Python |
| `package.json`, `tsconfig.json` | Node.js/TypeScript |
| `Cargo.toml` | Rust |
| `pom.xml`, `build.gradle`, `build.gradle.kts` | Java/Kotlin |
| `go.mod` | Go |
| `mix.exs` | Elixir |
| `Gemfile` | Ruby |
| `*.csproj`, `*.sln` | C#/.NET |

Read manifest contents to detect framework:
- Python: FastAPI? Django? Flask?
- Node.js: NestJS? Express? Hono?
- Rust: Axum? Actix-web? Rocket?
- Java: Spring Boot? Quarkus?

## Step 2: Confirm

Present detection results and ask for confirmation:
```
Detected backend stack:
  Language: {language}
  Framework: {framework}
  ORM: {orm}
  Validation: {validation}
  Migration: {migration}
  Test: {test framework}

Correct? (Y/n) or modify:
```

## Step 3: Generate

Create files in `.agents/skills/backend/stack/`:

### stack.yaml
```yaml
language: {language}
framework: {framework}
orm: {orm}
validation: {validation}
migration: {migration}
test: {test_framework}
source: detected
detected_from:
  - {manifest_file}
```

### tech-stack.md
Generate tech stack reference with these MANDATORY sections:
- Framework version and core API
- ORM/DB library and usage
- Validation library
- Migration tool
- Test framework
- Linter/formatter

### snippets.md
Generate copy-paste code patterns. MANDATORY patterns (all 8 required):
- [ ] Route/Handler + Auth example
- [ ] Validation Schema example
- [ ] ORM Model/Entity example
- [ ] DI (Dependency Injection) example
- [ ] Repository pattern example
- [ ] Paginated Query example
- [ ] Migration example
- [ ] Test example

### api-template.*
Generate CRUD endpoint boilerplate in the detected language.

### Host Project Domain Checklists (`docs/checklists/<domain>.md`)
Seed or update the host project's `docs/checklists/` (e.g., `docs/checklists/backend.md`, `docs/checklists/db.md`) with stack-specific starter negative constraints based on detected frameworks and libraries from Step 1:
- Framework-specific ORM anti-patterns (e.g., N+1 queries, unindexed foreign keys, lazy loading pitfalls)
- Timezone and datetime serialization patterns (e.g., UTC storage, explicit timezone parsing)
- Security and validation constraints (e.g., request schema validation, parameter binding, token handling)

Items must adhere to the standardized checklist regex format:
`- [ ] **{Topic}**: {Rule} (❌ Anti-pattern: `{X}`, ✅ Required: `{Y}`) [Ref: {file_or_incident}]`

## Step 4: Verify

Confirm generated files meet requirements:
- [ ] stack.yaml has language, framework, orm, validation fields
- [ ] snippets.md contains all 8 mandatory patterns
- [ ] tech-stack.md contains all 6 mandatory sections
- [ ] api-template file uses correct language extension
- [ ] `docs/checklists/<domain>.md` seeded with stack-specific negative constraints conforming to the checklist regex format
- [ ] Code follows existing project conventions

## Constraints

- Do NOT modify `.agents/skills/backend/SKILL.md` (abstract interface is protected)
- Do NOT modify `.agents/skills/stack-set/resources/` common files
- Only create/modify files in `.agents/skills/backend/stack/` and project `docs/checklists/`
- If `stack/` already exists, ask before overwriting
