# Workflow Guide - Examples

## Example 1: Full-Stack TODO App

**Input**: "Build a TODO app with JWT authentication"

**Workflow**:

```
Step 1: PM Agent plans the project
  -> 5 tasks: auth API, CRUD API, login UI, todo UI, QA review

Step 2: Spawn Priority 1 agents via OpenCode `task` tool
  # Run in parallel using multiple `task` tool calls in one message
  `task` tool: subagent_type="general", description="JWT auth API + TODO CRUD"
  `task` tool: subagent_type="general", description="Login/Register UI"

Step 3: Monitor progress
  - Use memory read tool to poll progress-{agent}.md files
  - Verify API contracts align between backend/frontend

Step 4: Spawn Priority 2 after P1 completes
  `task` tool: subagent_type="general", description="TODO List UI"

Step 5: Spawn Priority 3
  `task` tool: subagent_type="general", description="Security + Performance review"

Step 6: Address QA findings
  - Re-spawn agents for CRITICAL/HIGH issues
  - Verify fixes
```

## Example 2: Adding Feature to Existing App

**Input**: "Add a comment feature to the blog"

**Workflow**:

```
Step 1: PM Agent analyzes existing codebase and plans
  -> 2 tasks: Comments API, Comment Section UI

Step 2: Spawn Backend Agent first (API-first)
  `task` tool: subagent_type="general", description="Comments API with nested replies, pagination"

Step 3: After backend completes, spawn Frontend Agent
  `task` tool: subagent_type="general", description="Comment section UI using the new API endpoints"
  (Sequential because frontend depends on API contracts)

Step 4: QA review
  `task` tool: subagent_type="general", description="Security: XSS in comments, rate limiting; Performance: Pagination, N+1 queries"
```

## Example 3: When to Use Orchestrator Instead

**User**: "Run everything automatically"

**Response**:

```
"You want fully automated execution. Let me switch to the
orchestrator skill instead, which will:
1. Spawn agents automatically via the OpenCode `task` tool
2. Coordinate via progress files
3. Monitor progress and handle retries
4. Return unified results

Use coordination when you want step-by-step control.
Use orchestrator when you want hands-off automation."
```
