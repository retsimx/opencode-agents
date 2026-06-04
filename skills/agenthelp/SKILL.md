---
name: agenthelp
description: Agent help desk — lists all available skills/workflows with usage guidance and prints detailed information for any specific skill or workflow on request. Use for "what can you do", "help", "list skills", or asking about a specific skill or workflow.
---

# Agent Help Desk — Skills & Workflows Reference

## Scheduling

### Goal
Provide a quick-reference listing of all available agent skills and workflows with descriptions of what each does and when to use it. When the user asks about a specific skill or workflow by name, read its SKILL.md or workflow file and print the full details.

### Intent signature
- User asks "what can you do", "help", "list skills", "show workflows", "what skills exist"
- User mentions a specific skill name (e.g. "backend", "db", "debug", "deepsec")
- User mentions a specific workflow name (e.g. "orchestrate", "scm", "review")
- User asks "how do I use X" or "what does X do"

### When to use
- User asks for a general overview of available agent capabilities
- User asks about a specific named skill or workflow
- User wants to understand which skill to use for a particular task

### When NOT to use
- User has a concrete task that matches a domain skill -> route to that skill directly, do not load agenthelp
- User is asking about project architecture or configuration -> use architecture skill or relevant config
- User wants to execute a workflow -> use the relevant workflow directly

### Expected inputs
- A question about available skills, workflows, or a specific named one
- Optional: the name of a specific skill or workflow to drill into

### Expected outputs
- For general queries: a formatted table listing all skills and workflows with descriptions
- For specific queries: the full content of the relevant SKILL.md or workflow file with a summary

### Dependencies
- `.agents/skills/*/SKILL.md` files for skill descriptions
- `.agents/workflows/*.md` files for workflow descriptions

### Control-flow features
- Branches by whether user asks for overview or specific skill/workflow

## Structural Flow

### Entry
1. Parse the user's intent — are they asking for an overview of everything, or about a specific named item?
2. If overview: generate the skills table and workflows table from known index.
3. If specific `{name}`: check if it matches a skill or workflow, read the relevant file, and present it.

### Scenes
1. **PREPARE**: Classify user intent (overview vs specific).
2. **ACQUIRE**: Read relevant SKILL.md or workflow file if specific name given.
3. **ACT**: Print overview tables or full skill/workflow details.
4. **FINALIZE**: Offer to load a matched skill if the user wants to proceed.

### Transitions
- If the user asks about a skill by name that exists, read its `SKILL.md` and present the full content with a summary of what it does and when to use it.
- If the user asks about a workflow by name that exists, read the workflow `.md` file and present the full content.
- If the name doesn't match any skill or workflow, report what was found and list similar-sounding alternatives.
- After presenting specific skill info, ask if they'd like to load that skill to proceed.

### Failure and recovery
- If the named skill/workflow doesn't exist, list available names and suggest the closest match.
- If skill files are unreadable, print the known index from memory.

### Exit
- Success: user got the information they needed about available capabilities.
- Partial success: provided best-effort overview even if a specific file couldn't be read.

## Logical Operations

### Actions
| Action | SSL primitive | Evidence |
|--------|---------------|----------|
| Classify intent | `SELECT` | Overview vs specific name |
| Read skill file | `READ` | `.agents/skills/{name}/SKILL.md` |
| Read workflow file | `READ` | `.agents/workflows/{name}.md` |
| Print overview | `NOTIFY` | Formatted tables of all skills & workflows |
| Print specific details | `NOTIFY` | Full skill/workflow content |
| Suggest next step | `NOTIFY` | Ask if user wants to load the matched skill |

### Tools and instruments
- `read` for loading skill and workflow files
- `skill` for loading and activating a matched skill when user wants to proceed

### Canonical workflow path

**Overview listing:**
- List all skills with names and descriptions from known index.
- List all workflows with names and descriptions from known index.

**Specific skill drill-down:**
1. Read `.agents/skills/{name}/SKILL.md`.
2. Present the full file content to the user.
3. Ask if they want to load and use that skill.

**Specific workflow drill-down:**
1. Read `.agents/workflows/{name}.md`.
2. Present the full file content to the user.
3. Ask if they want to proceed with that workflow.

### Resource scope
| Scope | Resource target |
|-------|-----------------|
| `CODEBASE` | `.agents/skills/*/SKILL.md`, `.agents/workflows/*.md` |
| `LOCAL_FS` | Skill and workflow directories |
| `MEMORY` | Known index of all skills and workflows |

### Preconditions
- The `.agents/skills/` and `.agents/workflows/` directories exist.
- For specific lookups, the named skill or workflow file exists on disk.

### Effects and side effects
- Reads files from `.agents/skills/` and `.agents/workflows/`.
- May load a matched skill via `skill` tool if user wants to proceed.
- No destructive or persistent side effects.

### Guardrails
1. Always offer a next step after presenting specific skill details.
2. If the name doesn't match, suggest alternatives — don't just say "not found".
3. Don't load or execute a skill unless the user explicitly asks to proceed.

## References

Skills live in `.agents/skills/{name}/SKILL.md`. Workflows live in `.agents/workflows/{name}.md`.

### Available Skills

| Skill | Description |
|-------|-------------|
| **academic-writer** | Publication-grade English prose: essays, reports, literature reviews, anti-AI audits, rubric enforcement |
| **architecture** | Software/system design: module boundaries, tradeoff analysis, ATAM, CBAM, ADR decision records |
| **backend** | APIs, databases, auth: clean architecture (Repository/Service/Router pattern), REST, GraphQL |
| **brainstorm** | Design-first ideation: explore intent, constraints, approaches before planning |
| **coordination** | Multi-agent orchestration via CLI: step-by-step PM/Frontend/Backend/Mobile/QA coordination |
| **db** | SQL, NoSQL, vector DB: schema design, normalization, indexing, RAG retrieval, migration, capacity planning |
| **debug** | Bug diagnosis & fix: reproduce, root cause, minimal fix, regression tests, pattern scan |
| **deep-review** | Deterministic code review: correctness, regressions, system-level impact (analysis only, no QA checklist) |
| **deepinit** | Initialize AI harness: AGENTS.md TOC, ARCHITECTURE.md domain map, docs/ knowledge base |
| **deepsec** | Vercel deepsec vulnerability scanner: install, scan, triage, revalidate, CI gating |
| **design** | AI design system: DESIGN.md, typography, color, motion, responsive layouts, WCAG 2.2 |
| **dev-workflow** | Dev environment: mise tasks, git hooks, CI/CD, migrations, release automation |
| **docs** | Documentation drift detection: verify refs against codebase, propose sync patches |
| **frontend** | React, Next.js, TypeScript: FSD-lite architecture, shadcn/ui, design system alignment |
| **gh-autopilot** | GitHub issue → draft PR: fetch, plan, implement, review, verify CI, open PR from worktree |
| **image** | Multi-vendor AI image gen: Codex (gpt-image-2), Pollinations (flux/zimage) |
| **mobile** | Flutter, React Native: cross-platform mobile, Riverpod, widgets |
| **observability** | Traceability across layers: telemetry, APM, RUM, metrics, logs, SLO, incident forensics |
| **orchestrator** | Automated parallel agent spawning: MCP Memory coordination, progress monitoring |
| **pdf** | PDF → Markdown: opendataloader-pdf with text, tables, headings, images |
| **pm** | Product management: requirement decomposition, task breakdown, API contracts, prioritization |
| **qa** | Quality assurance: OWASP Top 10, performance, WCAG 2.1 AA, code quality, test coverage |
| **ralphreview** | Iterative deep-review loop: runs deep-review until 3 consecutive clean passes |
| **recap** | Session recap: analyze multi-tool conversation histories, generate themed summaries |
| **review** | Full QA pipeline: security, performance, accessibility, code quality pre-ship review |
| **scholar** | Academic research: Knows sidecar (.knows.yaml) generation, validation, paper authoring |
| **scm** | Git/SCM: branching, merges, conflicts, worktrees, Conventional Commits |
| **search** | Intent-based search router: Context7 docs, web search, gh/glab code search, Serena local |
| **skill-creator** | Skill authoring: create/update SSL-lite SKILL.md files |
| **stack-set** | Tech stack detection: scan manifests, generate stack.yaml, snippets, API boilerplate |
| **translator** | Context-aware translation: preserves tone, style, natural word order |
| **ultrawork** | 5-phase high-quality dev: PLAN → IMPL → VERIFY → REFINE → SHIP (11 review steps) |
| **work** | Multi-domain coordination: PM plan + parallel agents + QA review |

### Available Workflows

| Workflow | Description |
|----------|-------------|
| **architecture** | Diagnose architecture problems, compare options, produce ADR/recommendation |
| **brainstorm** | Explore intent, clarify constraints, produce approved design doc |
| **debug** | Structured bug diagnosis: reproduce, root cause, fix, regression tests |
| **deepinit** | Initialize AGENTS.md + ARCHITECTURE.md + docs/ knowledge base |
| **deepsec** | End-to-end deepsec vulnerability scanning and CI gating |
| **design** | Design systems, DESIGN.md, design tokens, accessibility checks |
| **docs** | Documentation drift detection and sync |
| **gh-autopilot** | GitHub issue → draft PR with auto-implement, review, CI verify |
| **orchestrate** | Automated parallel agent execution via OpenCode Task tool |
| **pdf** | PDF to Markdown conversion |
| **plan** | PM planning: requirements → prioritized tasks → machine-readable plan |
| **ralph** | Persistent review loop wrapping ultrawork with independent verifier |
| **recap** | Daily/period recap from multi-tool conversation histories |
| **review** | Full QA: security, performance, accessibility, code quality |
| **scm** | Git operations: branching, merge, conflict, Conventional Commits |
| **stack-set** | Auto-detect tech stack, generate stack-specific references |
| **ultrawork** | High-quality 5-phase development with deep quality gates |
| **work** | Multi-agent coordination with PM + parallel impl + QA |
