# Execution Protocol (Vendor-Agnostic)

When spawned via the OpenCode `task` tool, follow this protocol for shared state coordination.

## Key Practices

- **Use Task subagents for isolated work** — delegate distinct subtasks to
  sub-subagents rather than doing everything inline. Each subagent gets a
  focused context, reducing dilution and preventing scope creep.
- **Ask when uncertain** — use the `question` tool rather than making
  assumptions. Guessing leads to wasted work.
- **Stay in scope** — only work on your assigned task. Flag out-of-scope
  issues without fixing them.

## State Management

Use file-based I/O for coordination. Write results to `.agents/results/`.

### Path Resolution (CRITICAL)

All result, progress, and state files MUST be written to the **project root** `.agents/` directory, never to a subdirectory's `.agents/`.

- **Project root** = the git repository root (where `.git` exists)
- **Session-scoped naming**: when running under an orchestration session, append session ID as suffix:
  - `result-{agent-id}-{sessionId}.md` (e.g., `result-frontend-session-20260405-100835.md`)
  - `progress-{agent-id}-{sessionId}.md`
- **Manual (non-orchestrated) runs**: no suffix, `result-{agent-id}.md`

## On Start

1. Read `.agents/results/task-board.md` to confirm your assigned task
2. Create `.agents/results/progress-{agent-id}[-{sessionId}].md` with initial status

## During Execution

- Periodically update `progress-{agent-id}[-{sessionId}].md` with current state
- Include: action taken, current status, files created/modified

## On Completion

- Create `.agents/results/result-{agent-id}[-{sessionId}].md` with final result including:
  - Status: `completed` or `failed`
  - Summary of work done
  - Files created/modified
  - Acceptance criteria checklist

## On Failure

- Still create `result-{agent-id}[-{sessionId}].md` with Status: `failed`
- Include detailed error description and what remains incomplete
