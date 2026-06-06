# `.agents/` — Agent Skills & Configuration

> **Adapted fork of [oh-my-agent](https://github.com/first-fluke/oh-my-agent).**
> This directory provides a structured skill library and coding rules for the
> OpenCode agent framework. All skills are self-contained under `skills/`.

## Quick Start (New Project)

```bash
# 1. Clone into your project
git clone https://github.com/retsimx/opencode-agents.git .agents

# 2. OpenCode picks up skills automatically — just start using them

# 3. Grant the `task` tool to subagents in your opencode.json:
#    (required for orchestration workflows that spawn sub-subagents)
#    {
#      "agent": {
#        "general": { "permission": { "task": "allow" } },
#        "explore": { "permission": { "task": "allow" } }
#      }
#    }

# 4. Initialize AI harness (AGENTS.md, ARCHITECTURE.md, docs/)
opencode "run deepinit"

# 4. Detect tech stack and generate references
opencode "run stack-set"
```

OpenCode loads skills from `.agents/skills/` on startup — no additional config needed.

> **Important:** Add `.agents/` to your host project's `.gitignore`. This directory
> is its own standalone git repo — it should not be nested inside another.

## Directory Structure

```
.agents/
├── README.md              # This file
├── rules/                 # Domain-specific coding rules (i18n, etc.)
├── skills/                # Agent skill library (34 skills)
│   ├── _shared/           # Shared runtime protocols and core resources
│   ├── agenthelp/         # Help desk — list all skills on request
│   ├── backend/           # Backend API & server implementation
│   ├── frontend/          # Frontend UI implementation
│   ├── ...                # (see full list below)
│   └── work/              # Multi-agent coordination
```

## Initial Setup

Before using the agent system, consider running these setup skills:

| Skill | Purpose | When |
|-------|---------|------|
| `deepinit` | Initialize AI harness: AGENTS.md, ARCHITECTURE.md, structured docs/ | First time setting up a project |
| `stack-set` | Auto-detect tech stack, generate stack.yaml, snippets, API boilerplate | After deepinit, or when stack changes |

## Core Skills

Load skills via the `skill` tool by name (e.g., `skill "brainstorm"`).

### Planning & Design
| Skill | What it does |
|-------|-------------|
| `brainstorm` | Design-first ideation — explores intent, constraints, approaches |
| `plan` | PM-driven task breakdown with priorities and dependencies |

### Multi-Agent Execution
| Skill | What it does |
|-------|-------------|
| `orchestrate` | Spawns parallel subagents, file-based progress tracking |
| `work` | PM plan + parallel agents + QA review |
| `ultrawork` | 5-phase process with 11+ review steps |

### Review & Quality
| Skill | What it does |
|-------|-------------|
| `ralph` | Execution loop: ultrawork + independent verifier verification |
| `ralphreview` | Iterative deep-review loop until 3 clean passes |
| `review` | Full QA: security (OWASP), performance, accessibility, code quality |

### Operations
| Skill | What it does |
|-------|-------------|
| `scm` | Branch, merge, commit with Conventional Commits |
| `debug` | Structured diagnosis: reproduce → root cause → fix → regression test |
| `docs` | Documentation drift detection and sync |
| `recap` | Session recap from multi-tool conversation histories |

### Full skill list

Run the `agenthelp` skill for a complete listing with descriptions.

## Skills — When to Use Which

All 34 skills can be loaded via the `skill` tool. The main categories:

### Implementation Domains
- **backend** — APIs, databases, auth (Repository/Service/Router pattern)
- **frontend** — React, Next.js, TypeScript, shadcn/ui
- **mobile** — Flutter, React Native, cross-platform
- **db** — SQL, NoSQL, vector DB modeling, migrations

### Quality & Analysis
- **debug** — Bug diagnosis, root cause analysis, regression tests
- **qa** — Security, performance, accessibility, code quality
- **review** — Full pre-ship QA pipeline
- **deep-review** — Deterministic code correctness analysis
- **ralphreview** — Iterative review loop

### Process & Management
- **pm** — Product management, requirement decomposition, task breakdown
- **architecture** — Software design, tradeoff analysis, ADR records
- **design** — AI design systems, typography, color, motion
- **orchestrate** — Multi-agent spawning and coordination

### Specialized
- **scholar** — Academic research, paper sidecar generation
- **search** — Intent-based search across docs, web, code
- **scm** — Git operations, Conventional Commits
- **image** — Multi-vendor AI image generation
- **pdf** — PDF to Markdown conversion
- **translator** — Context-aware translation
- **docs** — Documentation drift detection
- **observability** — Traceability, telemetry, metrics, logs, SLO
- **deepsec** — Vulnerability scanning
- **deepinit** — Project AI harness initialization
- **dev-workflow** — Monorepo task automation, CI/CD, releases
- **stack-set** — Tech stack detection and reference generation
- **gh-autopilot** — GitHub issue → draft PR automation
- **skill-creator** — Authoring new skills in SSL-lite format
- **recap** — Conversation history analysis and summaries

### Meta
- **agenthelp** — Help desk: lists all skills on request
- **coordination** — Manual step-by-step multi-agent coordination
- **work** — Multi-domain feature coordination
- **ultrawork** — High-stakes 5-phase development
- **brainstorm** — Design-first ideation
- **academic-writer** — Publication-grade English prose

## Available Tools

The agent environment provides these built-in tools:

| Tool | Purpose |
|------|---------|
| **skill** | Load a skill's instructions into context |
| **Task** | Spawn a subagent for independent work |
| **bash** | Execute shell commands |
| **read** | Read files or directories |
| **write** | Write files |
| **edit** | Edit files with exact-string replacement |
| **grep** | Search file contents with regex |
| **glob** | Find files by glob pattern |
| **webfetch** | Fetch URL content |
| **todowrite** | Track task progress |
| **question** | Ask the user for input |

Load the `agenthelp` skill to get a full listing of all available skills with descriptions.
