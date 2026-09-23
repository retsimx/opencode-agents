# Coordination Protocol (File-Based)

When spawned as a subagent via the OpenCode `task` tool, use this protocol for
shared state coordination using the project's built-in `read`, `write`, `edit`
tools against `.agents/results/`.

## Guiding Principles

### Prefer Task subagents for isolated work
Delegate distinct subtasks to sub-subagents via the Task tool rather than doing
everything inline. Each subagent gets a focused context, reducing dilution and
preventing scope creep. Subagents are cheap — use them liberally.

**Note:** The `task` tool must be explicitly allowed in your OpenCode
configuration for subagents. Add to `opencode.json`:
```json
{
  "agent": {
    "general": { "permission": { "task": "allow" } },
    "explore": { "permission": { "task": "allow" } }
  }
}
```

### Ask when uncertain
Use the `question` tool whenever you face ambiguity. Never make assumptions —
guessing leads to wasted work and incorrect results. It's better to ask a quick
question than to build the wrong thing.

## Path Resolution & File-First State I/O

This library lives at `.agents/` inside a **host project**. Agents run with cwd = that host project root.

All coordination files MUST be written to **`.agents/results/`** (host-project-relative).
Skill and shared docs are loaded from **`.agents/skills/...`** (never skill-relative `../_shared/...`).

### Standard Deliverable Naming Formula

All subagent task deliverables follow the 4-part formula:
```
.agents/results/{type}-{role}-{taskSlug}-{sessionId}[-{index}].md
```
- `{type}`: Artifact type (`result`, `progress`, `review`, `adr`, `test`, `benchmark`).
- `{role}`: Subagent role (`backend`, `frontend`, `qa`, `reviewer`, `pm`, `designer`, etc.).
- `{taskSlug}`: Task slug (e.g., `auth-jwt`, `cart-api`, `checkout-flow`).
- `{sessionId}`: Session identifier (resolved via priority below).
- `[-{index}]`: Optional numeric sequence suffix for multi-part deliverables.

### Session ID Resolution Priority

When orchestrators or subagents resolve `SESSION_ID`, follow this strict priority order:
1. **Issue / Epic Slug**: If executing against a tracked issue or feature ticket (e.g., `issue-104`, `epic-booking-pdf`), use the normalized issue slug.
2. **Conversation Prefix**: If running in an interactive conversation session without an issue slug, use the first 8 characters of the conversation ID (e.g., `conv-97e0b488`).
3. **Timestamp Fallback**: If neither is available, use the literal `session`.

Then ALWAYS append a run suffix: `SESSION_ID = <slug>-<YYYYMMDD-HHMMSS>-<rand4hex>` (e.g. `issue-104-20260923-131500-a3f9`). Generate `<rand4hex>` with `openssl rand -hex 2`, or the portable fallback `head -c2 /dev/urandom | od -An -tx1 | tr -d ' \n'`. The suffix guarantees uniqueness when two sessions share a slug or start within the same second. The slug MUST be an issue/epic slug, a conversation prefix, or the literal `session` — never a skill name. A skill with an existing run identifier (e.g. the gardener family's `run-id`) satisfies `<sessionId>`; treat them as equivalent. The path shape may also be `<skill>/<sessionId>/` (as gardener-tend uses) instead of `<skill>-<sessionId>/`.

**Placeholder vs shell variable**: `{sessionId}` (equivalently `<sessionId>`) is the PLACEHOLDER used in artifact paths, templates, and state schemas. `$SESSION_ID` / `${SESSION_ID}` is the SHELL variable. Both denote the same resolved value. Never use the lowercase shell form `${sessionId}`. In subagent prompt templates, `<$VAR>` (e.g. `<$SESSION_ID>`, `<$RUN_TMP>`, `<$RESULTS_DIR>`, `<$PARENT_REPO>`) denotes a recorded absolute value that the orchestrator substitutes before dispatch.

### Session Artifact Naming & Run Temp (NORMATIVE)

- **Artifact filenames**: every session-scoped artifact MUST include `sessionId` immediately before its extension — `<base>-<sessionId>.<ext>` — or, for multi-part deliverables, before the optional sequence suffix — `<base>-<sessionId>-<index>.<ext>` (matching `...-{sessionId}[-{index}].md`). Examples: `result-qa-alignment-<sessionId>.md`, `plan-<sessionId>.json`, `task-board-<sessionId>.md`. A session artifact with a fixed, non-suffixed name is FORBIDDEN: two concurrent sessions must never share a filename.
- **Shared-within-session**: `task-board-<sessionId>.md`, `orchestrator-session-<sessionId>.md`, `session-ultrawork-<sessionId>.md`, `session-work-<sessionId>.md`, `session-ralph-<sessionId>.md`, `experiment-ledger-<sessionId>.md`, `session-metrics-<sessionId>.md`, and `subagent-ledger-<sessionId>.json` are each shared by all agents of ONE session and unique across sessions. Never use the unsuffixed forms.
- **Run temp directory (`RUN_TMP`)**: all inter-agent scratch I/O MUST live under a session-scoped directory `PARENT_REPO=$(git rev-parse --show-toplevel); RUN_TMP="$PARENT_REPO/.agents/results/tmp/<skill>-<sessionId>/"` — an ABSOLUTE path anchored to the control/parent repo — passed to subagents by reference (e.g. `$RUN_TMP/commit-msg.txt`, `$RUN_TMP/pr-body.txt`). Files INSIDE `RUN_TMP` need not repeat `sessionId` — the directory itself is already session-unique. This `results/tmp/` scratch directory is the explicit exception to the "no subdirectory" rule for result/state files. Global fixed-name temp paths such as `/tmp/pr-body.txt` are FORBIDDEN — they collide across concurrent runs (including across repositories) and are a shared-write hazard. `$(mktemp)` is permitted ONLY for single-command scratch where no session context exists; when a session context exists, use `RUN_TMP`.
- **Retention**: `RUN_TMP` and all session artifacts are RETAINED after completion as an immutable audit trail (they are session-unique and cannot collide). Do NOT auto-delete results, session files, or scratch; there is no automatic purge. (Narrow exception: the gardener family may prune stale git worktrees and prior-run temp directories under `$CONTROL_ROOT` — that is worktree GC, not deletion of session artifacts.)

### Orchestrator Zero-Context Relay Protocol (Pass-by-Reference)

To prevent orchestrator context degradation, orchestrators MUST operate on a **pass-by-reference** model:
1. **Never Ingest Full Artifacts**: The orchestrator must NOT read full subagent result files into its context window.
2. **Downstream Injection**: When a downstream subagent (e.g. QA, Reviewer, dependent Implementation agent) requires outputs from an upstream task, the orchestrator passes the upstream artifact file path (`file:///.../.agents/results/{type}-{role}-{taskSlug}-{sessionId}.md`) as a reference in `UPSTREAM_ARTIFACTS`.
3. **Direct Subagent Ingestion**: Downstream subagents view and parse upstream deliverable files directly in their isolated contexts.
4. **4-Line Ingestion Only**: The orchestrator context receives ONLY the concise 4-line chat completion summary from each subagent.

---

## On Start

1. Confirm `SESSION_ID`, `TASK_SLUG`, and designated `OUTPUT_FILE` from prompt (or apply Standalone Fallback).
2. Read `.agents/results/task-board-{sessionId}.md` (or task prompt) to confirm requirements.
3. Write initial status to `.agents/results/progress-{role}-{taskSlug}-{sessionId}.md`.

## During Execution

- Every 3-5 turns, update `.agents/results/progress-{role}-{taskSlug}-{sessionId}.md` to append a new turn entry.
- Include: action taken, current status, files created/modified, blockers.

## On Completion

1. Write exhaustive deliverables to designated `OUTPUT_FILE` (`.agents/results/{type}-{role}-{taskSlug}-{sessionId}[-{index}].md`) including:
   - Status: `SUCCESS` or `FAILED`
   - Complete technical summary of work done
   - Files created / modified / deleted
   - Acceptance criteria checklist with line-number citations
   - Domain checklist verification evidence with line-number citations
   - Test execution commands and outputs
2. Verify file write was successful on disk.
3. Return the standard 4-line chat return contract to the orchestrator:
   ```markdown
   ### Task Complete: {Role} — {Task Name}
   - **Status**: SUCCESS | BLOCKED | FAILED
   - **Summary**:
     - {Key outcome, finding, decision, or change 1}
     - {Key outcome, finding, decision, or change 2}
     - {Key outcome, finding, decision, or change 3}
   - **Artifact**: `file:///path/to/{output-file}.md`
   ```

## On Failure / Blocked

1. Still create `OUTPUT_FILE` with Status: `FAILED` or `BLOCKED` and detailed error logs, root cause analysis, and unfinished scope.
2. Return the standard 4-line chat return contract with Status: `FAILED` or `BLOCKED` and artifact link.

## Experiment Tracking (Optional Extension)

When a workflow activates Quality Score measurement, agents record experiments
in `.agents/results/experiment-ledger-{sessionId}.md`. After each measurable change, append
a row:

```
| # | Phase | Agent | Hypothesis | Score Before | Score After | Delta | Decision |
```

See `.agents/skills/_shared/conditional/experiment-ledger.md` for full format and analysis protocol.
