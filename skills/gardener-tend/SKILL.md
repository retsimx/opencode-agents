---
name: gardener-tend
description: >
  Tend open gardener PRs (`chore(gardener)` prefix): rebase, fix CI, act on
  reviewer comments, or promote draft to ready. One change per invocation, then
  exits. Works with GitHub (`gh`) and GitLab (`glab`). Family: gardener-sow →
  gardener-tend → gardener-harvest.
---

# Gardener Tend — PR Maintenance

## Scheduling

### Goal
Maintain PR health by checking merge status, diagnosing CI failures, and acting on reviewer comments. Makes **one change per invocation** in an isolated git worktree, then exits. The external loop schedules repeat invocations. Provider-agnostic: GitHub (`gh`) or GitLab (`glab`) per `../_shared/runtime/providers.md`.

### Intent signature
- User wants automated PR maintenance for gardener PRs
- User wants merge conflicts resolved automatically
- User wants CI failures diagnosed and fixed
- User wants reviewer comments acted upon
- User wants healthy draft PRs promoted to ready
- User invokes `/gardener-tend` or asks to tend gardener PRs

### When to use
- Periodic PR health maintenance (cron or on-demand)
- After pushing changes, to check if PRs need follow-up
- Batch-fixing multiple PRs via external loop
- Resolving rebase conflicts in `chore(gardener)` PRs

### When NOT to use
- Single PR with known issue (use `debug` skill)
- Ready-to-merge PR (no failures/comments/conflicts)
- Human-driven code review (use `review` skill)
- Creating new micro-fix PRs (use `gardener-sow`)
- Assessing/merging open PRs (use `gardener-harvest`)

### Expected inputs
- Repository with `gh` (GitHub) or `glab` (GitLab) CLI authenticated for `origin`
- `AGENTS.md` and `TESTING.md` present
- Main checkout may be dirty — skill does not require clean `main`

### Expected outputs
- **One** PR updated per invocation (rebase, CI fix, comment action, or draft promotion)
- No orphaned worktrees
- Main checkout state unchanged
- Resolved reviewer comments deleted after fix is pushed

### Dependencies
- Forge CLI (`gh` or `glab`) for PR queries, checks/pipelines, and comments — see `../_shared/runtime/providers.md`
- How to run (outer loops): `../_shared/runtime/gardener-running.md`
- `scm` skill for commit conventions
- `debug` skill for CI failure diagnosis
- Resources: `resources/worktree-isolation.md`, `resources/execution-protocol.md`
- Local test/lint/coverage commands from project

### Control-flow features
- One change per invocation — exits after first action taken
- Priority-ordered checks: merge status > CI failures > comments > draft promotion
- Worktree isolation — main checkout never modified
- Filters to `chore(gardener)` PRs only
- Provider detection at Entry; all forge calls use the shared command map

## Structural Flow

### Entry
1. Detect `PROVIDER` from `git remote get-url origin` per `../_shared/runtime/providers.md`. Verify CLI auth.
2. Fetch all open PRs with `chore(gardener)` title prefix (normalize fields per `providers.md`).
3. Order by age (oldest first).
4. For each PR, run checks in priority order.

### Scenes
1. **CHECK_MERGE**: Normalize `mergeableState` via View single PR. If conflicts or behind → rebase in worktree → push → exit.
2. **CHECK_CI**: Fetch CI/pipeline status. If any failed → create worktree → diagnose + fix all failures → verify locally → push → exit.
3. **CHECK_COMMENTS**: Fetch reviewer comments/notes. If actionable (code snippet) → implement change → delete comment → push → exit. If vague → ask for clarification → exit.
4. **CHECK_DRAFT**: If PR is draft + all CI complete + no failures + no comments → promote to ready → exit.
5. **ALL_CLEAR**: If no changes across all PRs → exit silently.

### Transitions
- If one PR needs action → act → exit. Next invocation continues from next PR.
- If a check produces multiple diagnosis (e.g. multiple CI failures) → fix all in one pass.
- If a fix fails local verification → fix it (local verification cannot fail by design).

### Failure and recovery
- If provider CLI is not authenticated → exit with error message.
- If worktree creation fails → skip PR and try next.
- If push fails (force-push rejected) → fetch latest and retry.
- If comment is ambiguous → add reply asking for clarification, do not delete.

### Exit
- Success: one PR updated (rebase, fix, comment, or promotion).
- No-op: all PRs healthy → exit silently.
- Error: blocker documented.

## Logical Operations

### Actions
| Action | SSL primitive | Evidence |
|--------|---------------|----------|
| Detect provider | `READ` | `git remote get-url origin` |
| Fetch open PRs | `CALL_TOOL` | List open PRs (`providers.md`) |
| Check merge state | `CALL_TOOL` | View single PR → normalize `mergeableState` |
| Check CI status | `CALL_TOOL` | Fetch CI / pipeline status |
| Fetch comments | `CALL_TOOL` | Reviewer comments / notes |
| Create worktree | `CALL_TOOL` | `git worktree add` |
| Rebase branch | `CALL_TOOL` | `git rebase` + conflict resolution |
| Diagnose CI fail | `CALL_TOOL` + `INFER` | Read CI logs + fix |
| Run local verify | `VALIDATE` | Test/lint/coverage commands |
| Push changes | `CALL_TOOL` | `git push --force-with-lease` |
| Delete comment | `CALL_TOOL` | Delete comment/note (`providers.md`) |
| Promote draft | `CALL_TOOL` | Promote draft → ready |
| Clean up worktree | `CALL_TOOL` | `git worktree remove` |

### Canonical workflow path

```
1. Detect PROVIDER; auth check
2. List open PRs; filter title startswith "chore(gardener)"; sort createdAt asc
3. Per PR: normalize mergeableState → CI → comments → draft promotion
4. All forge commands from ../_shared/runtime/providers.md for PROVIDER
```

### Resource scope
| Scope | Resource target |
|-------|-----------------|
| `CODEBASE` | PR source code, tests, CI config |
| `LOCAL_FS` | Git worktrees for isolated changes |
| `PROCESS` | `gh` / `glab`, test/lint/coverage commands |
| `MEMORY` | PR state, diagnosis notes, verification evidence |
| `NETWORK` | GitHub or GitLab API via forge CLI |

### Preconditions
- Matching forge CLI is authenticated and has access to the repository.
- Test, lint, and coverage commands are available locally.
- Main checkout may be in any state (worktrees used for all changes).

### Effects and side effects
- Pushes force-pushed commits to PR branches.
- Deletes resolved reviewer comments/notes.
- Promotes draft PRs to ready for review.
- Creates and removes git worktrees.

### Guardrails
1. **chore(gardener) only** — never operate on non-gardener PRs
2. **One change per invocation** — rebase, fix, comment, or promote, never mixed
3. **Never push failing CI** — always verify locally before push
4. **Delete comments atomically** — only after push succeeds
5. **Worktree isolation** — all changes in worktrees, never main repo
6. **Fix all CI failures in one pass** — diagnose and fix every failed step
7. **Ask when vague** — ambiguous comments get a reply, not a delete
8. **Age priority** — oldest PRs first
9. **Provider-agnostic** — never hardcode `gh` or `glab`; use `providers.md`

## References

- How to run (outer loops): `../_shared/runtime/gardener-running.md`
- Sibling skills: `gardener-sow` (create PRs), `gardener-harvest` (merge queue)
- Provider CLI map: `../_shared/runtime/providers.md`
- Worktree isolation: `resources/worktree-isolation.md`
- Execution protocol: `resources/execution-protocol.md`
- Context loading: `../_shared/core/context-loading.md`
- Reasoning templates: `../_shared/core/reasoning-templates.md`
