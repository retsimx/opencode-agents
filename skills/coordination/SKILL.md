---
name: coordination
description: Guide for coordinating PM, Frontend, Backend, Mobile, and QA agents on complex projects via CLI. Use for manual step-by-step coordination and workflow guidance.
---

# Multi-Agent Workflow Guide

## Scheduling

### Goal
Guide manual multi-agent coordination for complex work that spans PM, frontend, backend, mobile, and QA responsibilities.

### Intent signature
- User wants step-by-step coordination, manual agent spawning, or multi-domain work planning without full automation.
- Task spans multiple specialist agents and requires contract alignment.

### When to use

- Complex feature spanning multiple domains (full-stack, mobile)
- Coordination needed between frontend, backend, mobile, and QA
- User wants step-by-step guidance for multi-agent coordination

### When NOT to use

- Simple single-domain task -> use the specific agent directly
- User wants automated execution -> use orchestrate
- Quick bug fixes or minor changes

### Expected inputs
- Complex feature or project goal
- Required domains and priority tiers
- Workspace/session constraints and API/data contract needs

### Expected outputs
- Manual coordination sequence
- PM task decomposition, agent spawn order, monitoring guidance, and QA review step
- API/data contract alignment checkpoints

### Dependencies
- PM, frontend, backend, mobile, QA, and orchestrate skills
- `resources/examples.md`
- OpenCode `task` tool and progress/result memory conventions

### Control-flow features
- Branches by task complexity, priority tiers, dependency ordering, and whether automation is desired
- Spawns independent same-priority tasks in parallel when appropriate
- Monitors progress files and contract alignment

## Structural Flow

### Entry
1. Confirm the task is complex enough for multi-agent coordination.
2. Start with PM task decomposition.
3. Identify priority tiers and shared contracts.

### Scenes
1. **PREPARE**: Define session, domains, and task decomposition needs.
2. **ACT**: Spawn agents by priority with separate workspaces.
3. **VERIFY**: Monitor progress and API/data contract alignment.
4. **FINALIZE**: Run QA review and coordinate remediation.

### Transitions
- If task is simple, route to one specialist.
- If user wants automated execution, use orchestrate.
- If QA finds CRITICAL issues, re-spawn responsible agents.

### Failure and recovery
- If contracts diverge, pause downstream frontend/mobile work until backend/API contract is reconciled.
- If agent workspaces conflict, split ownership boundaries.
- If progress stalls, inspect progress files and reissue focused instructions.

### Exit
- Success: specialist outputs are coordinated and QA-reviewed.
- Partial success: blocked agents, contract conflicts, or QA failures are explicit.

## Logical Operations

### Actions
| Action | SSL primitive | Evidence |
|--------|---------------|----------|
| Read request and domains | `READ` | User prompt and project context |
| Select agent plan | `SELECT` | PM decomposition and priority tiers |
| Spawn agents | `CALL_TOOL` | OpenCode `task` tool (subagent_type="general") |
| Monitor progress | `READ` | `progress-{agent}.md` |
| Validate contracts | `VALIDATE` | API/data model alignment |
| Notify coordination status | `NOTIFY` | Final coordination summary |

### Tools and instruments
- OpenCode `task` tool (subagent_type="general"/"explore"), PM/frontend/backend/mobile/QA agents
- Memory/progress/result files
- OpenCode built-in tools (`grep`, `glob`, `read`, `write`, `edit`) for exploration and progress tracking when used by specialists

### Canonical task pattern
```
`task` tool invocation pattern:
- PM: Call Task with subagent_type="general", prompt=<PM task>, and workdir=<workspace>/pm
- Backend: Call Task with subagent_type="general", prompt=<backend task>, and workdir=<workspace>/backend (run in parallel)
- Frontend: Call Task with subagent_type="general", prompt=<frontend task>, and workdir=<workspace>/frontend (run in parallel)
```

### Resource scope
| Scope | Resource target |
|-------|-----------------|
| `LOCAL_FS` | Progress/result files and workspaces |
| `PROCESS` | `task` tool invocations |
| `MEMORY` | Session state and task board |
| `CODEBASE` | Shared contracts and implementation areas |

### Preconditions
- Task requires multiple domains.
- PM decomposition can identify independent priority tiers.

### Effects and side effects
- Spawns or guides multiple agents.
- Coordinates workspace ownership and QA feedback.

### Guardrails

1. Always start with PM Agent for task decomposition
2. Spawn independent tasks in parallel (same priority tier)
3. Define API contracts before frontend/mobile tasks
4. QA review is always the final step
5. Assign separate workspaces to avoid file conflicts
6. Use OpenCode's built-in tools (`grep`, `glob`, `read`) for code exploration
7. Never skip steps in the workflow; follow each step sequentially without omission

### Workflow

### Step 1: Plan with PM Agent

PM Agent analyzes requirements, selects tech stack, creates task breakdown with priorities.

### Step 2: Spawn Agents by Priority

Use the OpenCode `task` tool for each task:

1. Use Task with subagent_type="general" for each task, passing the agent role in the prompt
2. Spawn all same-priority tasks in parallel via concurrent `task` tool calls
3. Assign separate workspaces via the workdir parameter to avoid file conflicts

### Step 3: Monitor & Coordinate

- Use memory read tool to poll `progress-{agent}.md` files
- Verify API contracts align between agents
- Ensure shared data models are consistent

### Step 4: QA Review

Spawn QA Agent last to review all deliverables. Address CRITICAL issues by re-spawning agents.

### Automated Alternative

For fully automated execution without manual spawning, use the **orchestrate** skill instead.

## References

- Workflow examples: `resources/examples.md`
