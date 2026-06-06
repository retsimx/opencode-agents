---
name: agenthelp
description: Agent help desk — lists all available skills with usage guidance and prints detailed information for any specific skill on request. Use for "what can you do", "help", "list skills", or asking about a specific skill.
---

# Agent Help Desk — Skills Reference

## Scheduling

### Goal
Provide a quick-reference listing of all available agent skills with descriptions of what each does and when to use it. When the user asks about a specific skill by name, load it via the `skill` tool and present the full details.

### Intent signature
- User asks "what can you do", "help", "list skills"
- User mentions a specific skill name (e.g. "backend", "db", "debug")
- User asks "how do I use X" or "what does X do"

### When to use
- User asks for a general overview of available agent capabilities
- User asks about a specific named skill
- User wants to understand which skill to use for a particular task

### When NOT to use
- User has a concrete task that matches a domain skill -> route to that skill directly, do not load agenthelp
- User is asking about project architecture or configuration -> use architecture skill or relevant config

### Expected inputs
- A question about available skills, or a specific named one
- Optional: the name of a specific skill to drill into

### Expected outputs
- For general queries: a formatted table listing all skills with descriptions
- For specific queries: the full content of the relevant SKILL.md with a summary

### Dependencies
- `.agents/skills/*/SKILL.md` files for skill descriptions

### Control-flow features
- Branches by whether user asks for overview or specific skill

## Structural Flow

### Entry
1. Parse the user's intent — are they asking for an overview of everything, or about a specific named item?
2. If overview: print the skills table from known index.
3. If specific `{name}`: load the skill via the `skill` tool and present it.

### Scenes
1. **PREPARE**: Classify user intent (overview vs specific).
2. **ACT**: Print overview table or load and present specific skill.
3. **FINALIZE**: Offer to load a matched skill if the user wants to proceed.

### Transitions
- If the user asks about a skill by name that exists, load it via the `skill` tool and present the content.
- If the name doesn't match any skill, report what was found and list similar-sounding alternatives.
- After presenting specific skill info, ask if they'd like to load that skill to proceed.

### Failure and recovery
- If the named skill doesn't exist, list available names and suggest the closest match.
- If skill files are unreadable, print the known index from memory.

### Exit
- Success: user got the information they needed about available capabilities.

## Logical Operations

### Actions
| Action | SSL primitive | Evidence |
|--------|---------------|----------|
| Classify intent | `SELECT` | Overview vs specific name |
| Print overview | `NOTIFY` | Formatted table of all skills |
| Load specific skill | `CALL_TOOL` | `skill` tool with skill name |
| Suggest next step | `NOTIFY` | Ask if user wants to load the matched skill |

### Tools and instruments
- `skill` for loading and activating a matched skill when user wants to proceed

### Resource scope
| Scope | Resource target |
|-------|-----------------|
| `CODEBASE` | `.agents/skills/*/SKILL.md` |
| `LOCAL_FS` | Skill directory |
| `MEMORY` | Known index of all skills |

### Preconditions
- The `.agents/skills/` directory exists.

### Effects and side effects
- Reads files from `.agents/skills/`.
- May load a matched skill via `skill` tool if user wants to proceed.

### Guardrails
1. Always offer a next step after presenting specific skill details.
2. If the name doesn't match, suggest alternatives — don't just say "not found".
3. Don't load or execute a skill unless the user explicitly asks to proceed.

## Available Skills

| Skill | Description |
|-------|-------------|
| **academic-writer** | Publication-grade English prose: essays, reports, literature reviews, anti-AI audits, rubric enforcement |
| **architecture** | Software/system design: module boundaries, tradeoff analysis, ATAM, CBAM, ADR decision records |
| **backend** | APIs, databases, auth: clean architecture (Repository/Service/Router pattern), REST, GraphQL |
| **brainstorm** | Design-first ideation: explore intent, constraints, approaches before planning |
| **coordination** | Multi-agent coordination: step-by-step PM/Frontend/Backend/Mobile/QA coordination |
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
| **orchestrate** | Automated parallel agent execution: spawns subagents via task tool, file-based progress tracking |
| **pdf** | PDF → Markdown: opendataloader-pdf with text, tables, headings, images |
| **plan** | PM planning: requirements → prioritized tasks → machine-readable plan + human tracker |
| **pm** | Product management: requirement decomposition, task breakdown, API contracts, prioritization |
| **qa** | Quality assurance: OWASP Top 10, performance, WCAG 2.1 AA, code quality, test coverage |
| **ralph** | Persistent execution loop: wraps ultrawork with independent verifier verification per iteration |
| **ralphreview** | Iterative deep-review loop: runs deep-review until 3 consecutive clean passes |
| **recap** | Session recap: analyze multi-tool conversation histories, generate themed summaries |
| **review** | Full QA pipeline: security, performance, accessibility, code quality pre-ship review |
| **scholar** | Academic research: Knows sidecar (.knows.yaml) generation, validation, paper authoring |
| **scm** | Git/SCM: branching, merges, conflicts, worktrees, Conventional Commits, commit governance |
| **search** | Intent-based search router: Context7 docs, web search, gh/glab code search, local file search |
| **skill-creator** | Skill authoring: create/update SSL-lite SKILL.md files |
| **stack-set** | Tech stack detection: scan manifests, generate stack.yaml, snippets, API boilerplate |
| **translator** | Context-aware translation: preserves tone, style, natural word order |
| **ultrawork** | 5-phase high-quality dev: PLAN → IMPL → VERIFY → REFINE → SHIP (11 review steps) |
| **work** | Multi-domain coordination: PM plan + parallel agents + QA review |
