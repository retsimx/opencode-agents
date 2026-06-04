# Coordination Protocol (File-Based)

When spawned as a subagent via the OpenCode Task tool, use this protocol for
shared state coordination using the project's built-in `read`, `write`, `edit`
tools against `.agents/results/`.

## Path Resolution

All coordination files MUST be written to the **project root** `.agents/results/`
directory. Session-scoped naming uses a session ID suffix:
- `result-{agent-id}-{sessionId}.md`
- `progress-{agent-id}-{sessionId}.md`
- Manual (non-orchestrated) runs: no suffix, `result-{agent-id}.md`

## On Start

1. Read `.agents/results/task-board.md` to confirm your assigned task
2. Write `.agents/results/progress-{agent-id}[-{sessionId}].md` with initial status

## During Execution

- Every 3-5 turns, edit `.agents/results/progress-{agent-id}[-{sessionId}].md`
  to append a new turn entry
- Include: action taken, current status, files created/modified

## On Completion

- Write `.agents/results/result-{agent-id}[-{sessionId}].md` with final result
  including:
  - Status: `completed` or `failed`
  - Summary of work done
  - Files created/modified
  - Acceptance criteria checklist

## On Failure

- Still create `result-{agent-id}[-{sessionId}].md` with Status: `failed`
- Include detailed error description and what remains incomplete

## Experiment Tracking (Optional Extension)

When a workflow activates Quality Score measurement, agents record experiments
in `.agents/results/experiment-ledger.md`. After each measurable change, append
a row:

```
| # | Phase | Agent | Hypothesis | Score Before | Score After | Delta | Decision |
```

See `../conditional/experiment-ledger.md` for full format and analysis protocol.
