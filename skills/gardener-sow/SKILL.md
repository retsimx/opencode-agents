---
name: gardener-sow
description: >
  Single-shot software-gardening sow: find one micro-improvement in an isolated
  git worktree, verify CI locally, open one small draft PR, then exit. No user
  interaction. Orchestration-only — all work runs in task subagents. Continuous
  runs use an external shell loop (see running docs). Family: gardener-sow →
  gardener-tend → gardener-harvest.
---

# Gardener Sow — Single Micro-Improvement

## Scheduling

### Goal
Improve the repository by **exactly one** micro-change per invocation. Runs in an isolated git worktree, produces one small human-reviewable draft PR (or skips/fails cleanly), cleans up the worktree, then **ends the turn**. Continuous gardening is the caller's outer shell loop — never an in-skill infinite loop (avoids context dilution).

### Intent signature
- User invokes `/gardener-sow` or asks to sow one gardener iteration
- User wants one small draft PR with full CI verification before push
- Continuous improvement via external `while` / cron wrapping this skill

### When to use
- One autonomous micro-improvement (or an outer loop of many)
- Background gardening with draft PRs for human review
- Cheap/local models where fresh context each shot matters

### When NOT to use
- Bounded task with defined completion criteria -> use `ralph`
- Review-only loop until clean -> use `ralphreview`
- Single issue with user approval gates -> use `issue-autopilot`
- One-shot review -> use `deep-review`
- Maintain existing `chore(gardener)` PRs -> use `gardener-tend`
- Assess/merge open PRs -> use `gardener-harvest`

### Expected inputs
- Repository with `gh` (GitHub) or `glab` (GitLab) CLI authenticated for `origin`
- `AGENTS.md` and `TESTING.md` present
- Optional: existing `.agents/results/gardener-state.md` from prior runs
- Main checkout may be dirty — gardener does not require or enforce a clean `main`

### Expected outputs
- At most one small draft PR (`gardener/iter-{NNNN}-{slug}`) on success
- Log row in `.agents/results/gardener-state.md` (Status and Description); rows removed once PR is merged
- No orphaned worktrees after the invocation
- Main checkout state unchanged
- Process exits after one attempt (success, skip, or failure)

### Dependencies
- OpenCode `task` tool — all scan, work, verify, ship, worktree, and revert work
- `scm` skill — loaded by SHIP subagent for commit conventions
- Shared provider map: `../_shared/runtime/providers.md` (`gh` / `glab`)
- How to run (outer loops): `../_shared/runtime/gardener-running.md`
- Resources: `resources/worktree-isolation.md`, `resources/worker-prompt.md`, `resources/worktree-prompt.md`, `resources/pr-size-limits.md`, `resources/exclusions.md`, `resources/ci-gates.md`, `resources/init-prompt.md`, `resources/verify-prompt.md`, `resources/ship-prompt.md`, `resources/revert-prompt.md`
- Forge CLI (`gh` or `glab`) for draft PR creation
- Design doc: `docs/plans/designs/gardener-loop.md`

### Control-flow features
- **Single shot** — one INIT→…→CLEANUP pass, then exit (no in-skill loop)
- Each invocation uses an isolated git worktree; main checkout never modified
- Mandatory worktree cleanup before exit (success or failure)
- Every subagent receives `MAIN_REPO` and `WORKTREE` paths
- `FATAL|main_modified` aborts the shot immediately
- No `question` tool — fully autonomous

## Structural Flow

### Entry
1. Verify this is an autonomous gardening session (no user prompts).
2. Record `MAIN_REPO` as absolute path to main repository checkout.
3. Initialize or read `.agents/results/gardener-state.md`.
4. Run the single pipeline once starting at INIT.

### Scenes
1. **INIT**: Task subagent (`init-prompt.md`) detects provider, verifies forge CLI auth, fetches `origin main`, reads state; does not check main cleanliness; returns `READY|<iteration>`.
2. **WORKTREE_SETUP**: Task subagent (`worktree-prompt.md`, MODE=SETUP) creates isolated worktree from `origin/main` (works even if main is dirty); returns `WORKTREE_READY|<path>|<branch>`.
3. **SCAN**: Task subagent (`worker-prompt.md`, MODE=SCAN) inside `WORKTREE`; returns `FOUND|...`, `SKIP|...`, or `FATAL|main_modified|...`.
4. **WORK**: Task subagent (`worker-prompt.md`, MODE=WORK) inside `WORKTREE`; returns `SUCCESS|...`, `FAIL|...`, `ABORT|...`, or `FATAL|main_modified|...`.
5. **VERIFY**: Task subagent (`verify-prompt.md`) inside `WORKTREE`; returns `PASS`, `FAIL:<reason>`, or `FATAL|main_modified|...`.
6. **SHIP**: Task subagent (`ship-prompt.md`) inside `WORKTREE`; returns `SHIPPED|<branch>|<pr_url>`, `SHIP_FAIL:<reason>`, or `FATAL|main_modified|...`.
7. **LOG**: SHIP subagent appends iteration row with Status=open; INIT removes rows whose PRs are merged, updates rows whose PRs are closed (Status=open → Status=closed), and removes rows with Status=done.
8. **WORKTREE_CLEANUP**: Task subagent (`worktree-prompt.md`, MODE=CLEANUP) removes worktree — **mandatory before exit**; does not alter main working tree.
9. **EXIT**: End the turn. Do not start another iteration.

### Transitions
- WORKTREE_SETUP fails -> LOG, exit (no cleanup needed).
- Any `FATAL|main_modified` -> WORKTREE_CLEANUP, LOG `FATAL`, exit.
- SCAN returns `SKIP` -> WORKTREE_CLEANUP, LOG `SKIP`, exit.
- WORK returns `FAIL` or `ABORT` -> revert via task, WORKTREE_CLEANUP, LOG, exit.
- VERIFY returns `FAIL` -> revert via task, WORKTREE_CLEANUP, LOG, exit.
- SHIP returns `SHIP_FAIL` -> revert via task, WORKTREE_CLEANUP, LOG, exit.
- SHIP returns `SHIPPED` -> LOG `PASS`, WORKTREE_CLEANUP, exit.
- Any error -> revert if needed, WORKTREE_CLEANUP (always), LOG, exit.

### Failure and recovery
| Failure | Recovery |
|---------|----------|
| `FATAL\|main_modified` | cleanup worktree, log `FATAL`, exit |
| VERIFY fails | revert via task, cleanup worktree, log `FAIL:<reason>`, exit |
| SHIP fails | revert via task, cleanup worktree, log `SHIP_FAIL`, exit |
| SCAN finds nothing | cleanup worktree, log `SKIP`, exit |
| Subagent timeout/error | revert if dirty in worktree, cleanup worktree, log `ERROR:<message>`, exit |
| CLEANUP fails | log `CLEANUP_FAIL`, exit anyway |

### Exit
- Always exit after one pipeline attempt (PASS / SKIP / FAIL / FATAL / ERROR).
- Outer shell loops re-invoke for the next shot with a fresh context.

## Logical Operations

### Actions
| Action | SSL primitive | Evidence |
|--------|---------------|----------|
| Read/init state | `READ` | `.agents/results/gardener-state.md` |
| Delegate init | `CALL_TOOL` | task subagent returns `READY` |
| Delegate worktree setup | `CALL_TOOL` | task subagent returns `WORKTREE_READY` |
| Delegate scan | `CALL_TOOL` | task subagent returns `FOUND` or `SKIP` |
| Delegate work | `CALL_TOOL` | task subagent returns `SUCCESS` or `FAIL` |
| Delegate verify | `CALL_TOOL` | task subagent returns `PASS` or `FAIL` |
| Delegate ship | `CALL_TOOL` | task subagent returns `SHIPPED` or `SHIP_FAIL` |
| Revert on failure | `CALL_TOOL` | task subagent per `revert-prompt.md` |
| Delegate worktree cleanup | `CALL_TOOL` | task subagent returns `CLEANED` |
| Append log | `UPDATE_STATE` | state file append with Status=open (SHIP subagent) |
| Remove merged row | `UPDATE_STATE` | state file row removal (INIT subagent — also removes Status=done rows) |
| Update closed row | `UPDATE_STATE` | state file Status=open → Status=closed (INIT subagent) |
| Exit | `UPDATE_STATE` | end turn after cleanup |

### Tools and instruments
- OpenCode `task` tool (`subagent_type="general"`) — **only** tool for substantive work
- Orchestrator may use `bash` **only** for: appending to state file, reading iteration counter
- Orchestrator **MUST NOT**: `read` source files, `edit`/`write` source files, run ruff/tests/coverage, run `git commit`/`git push`/forge PR create, create/remove worktrees

### Canonical workflow path

**CRITICAL: The orchestrator is a thin single-shot runner. It MUST NOT loop. It MUST NOT perform scan, work, verify, ship, or worktree operations inline. Every phase except LOG uses the built-in `task` tool. Never run `$ task` in bash.**

**Every task subagent prompt MUST include `MAIN_REPO` (absolute path) and `WORKTREE` (absolute path, once created). Nested subagents inherit both. See `resources/worktree-isolation.md`.**

**All git/forge/bash commands inside subagents MUST be wrapped with `timeout 300` to prevent infinite hangs.**

```
STATE_FILE = .agents/results/gardener-state.md
MAIN_REPO = absolute path to main repository checkout

worktree = null
branch = null

# INIT — resources/init-prompt.md
Call task (gardener-init) with MAIN_REPO
Return: READY|<iteration_number> or NOT_READY|<reason>
if NOT_READY: log, EXIT

# WORKTREE_SETUP — resources/worktree-prompt.md MODE=SETUP, SLUG=pending
Call task (gardener-worktree-setup) with MAIN_REPO, ITERATION, SLUG=pending
Return: WORKTREE_READY|<worktree_path>|<branch> or WORKTREE_FAIL|<reason>
if WORKTREE_FAIL: log, EXIT

worktree = path; branch = branch name

# SCAN — resources/worker-prompt.md MODE=SCAN
Call task (gardener-scan) with MAIN_REPO, WORKTREE
Return: FOUND|<slug>|... or SKIP|... or FATAL|main_modified|...

if FATAL or SKIP:
    Call task (gardener-worktree-cleanup) with MAIN_REPO, WORKTREE, BRANCH
    log; EXIT

# WORK
Call task (gardener-work) with MAIN_REPO, WORKTREE, ITEM=<slug>|<description>|<rationale>
Return: SUCCESS|... or FAIL|... or ABORT|... or FATAL|main_modified|...

if FATAL/FAIL/ABORT:
    Call task (gardener-revert) with MAIN_REPO, WORKTREE
    Call task (gardener-worktree-cleanup) with MAIN_REPO, WORKTREE, BRANCH
    log; EXIT

# VERIFY
Call task (gardener-verify) with MAIN_REPO, WORKTREE
Return: PASS or FAIL:<reason> or EXCLUDED:<path> or FATAL|main_modified|...

if not PASS or FATAL:
    Call task (gardener-revert) with MAIN_REPO, WORKTREE
    Call task (gardener-worktree-cleanup) with MAIN_REPO, WORKTREE, BRANCH
    log; EXIT

# SHIP
Call task (gardener-ship) with MAIN_REPO, WORKTREE, BRANCH, ITERATION, SLUG, DESCRIPTION
Return: SHIPPED|... or SHIP_FAIL|... or FATAL|main_modified|...

if SHIP_FAIL or FATAL:
    Call task (gardener-revert) with MAIN_REPO, WORKTREE
    Call task (gardener-worktree-cleanup) with MAIN_REPO, WORKTREE, BRANCH
    log; EXIT

# LOG — SHIP already appended the state row (Status=open)

# WORKTREE_CLEANUP — mandatory before exit
Call task (gardener-worktree-cleanup) with MAIN_REPO, WORKTREE, BRANCH
EXIT
```

### Resource scope
| Scope | Resource target |
|-------|-----------------|
| `LOCAL_FS` | `.agents/results/gardener-state.md`, ephemeral `../sarahwebsite-gardener-{NNNN}` worktrees |
| `PROCESS` | task subagent processes |
| `CODEBASE` | modified only by subagents inside worktrees, never main checkout or orchestrator |

### Preconditions
- Matching forge CLI (`gh` or `glab`) installed and authenticated for `origin`
- `poetry` environment available in worktree
- WORKTREE_SETUP creates branch from `origin/main` via `git worktree add` (main may be dirty)

### Effects and side effects
- Subagents may modify code inside worktree, run tests, commit, push, create draft PRs
- Orchestrator only appends to state file
- At most one new branch and one draft PR per successful invocation
- Worktree removed before exit; main checkout state preserved
- Never commits to `main`

### Guardrails
1. **Orchestrator never edits source** — no `read`/`edit`/`write` on application code
2. **Orchestrator never runs CI** — ruff, tests, coverage run only in subagents
3. **No `question` tool** — fully autonomous, no user interaction
4. **Single shot only** — never loop inside the skill; outer shell owns continuity
5. **One fix per invocation** — one commit, one draft PR, one specific concern
6. **Never commit to or modify `main`** — `FATAL|main_modified` on violation
7. **Isolated worktree per invocation** — all SCAN/WORK/VERIFY/SHIP inside `WORKTREE` only
8. **Every subagent gets `MAIN_REPO` + `WORKTREE`** — nested subagents inherit both
9. **Dirty main is allowed** — worktree created from `origin/main` without cleaning main
10. **Mandatory cleanup** — worktree removed before exit regardless of outcome
11. **Human-reviewable size** — see `resources/pr-size-limits.md` (≤150 diff lines, one concern; no hard file limit)
12. **Respect exclusions** — see `resources/exclusions.md`
13. **Tests must pass** before SHIP — coverage is informational, no hard percentage
14. **Draft PRs only** — always create via the Create draft PR command for `$PROVIDER` in `../_shared/runtime/providers.md`
15. **All PRs via forge CLI** — `gh` or `glab` per `providers.md`; never use web UI or other tools
16. **Nested tasking allowed** — subagents may spawn further subagents (must pass paths)
17. **AGENTS.md and TESTING.md** — subagents must comply
18. **State file tracks open/closed rows** — SHIP appends new row with Status=open, INIT removes merged rows, updates closed rows to Status=closed, and removes Status=done rows
19. **Description column for SCAN** — SCAN compares slug AND description to distinguish similar fixes
20. **Provider-agnostic** — detect `PROVIDER` once; pass to every subagent; never hardcode `gh` or `glab`

## References
- How to run (outer loops): `../_shared/runtime/gardener-running.md`
- Sibling skills: `gardener-tend` (maintain PRs), `gardener-harvest` (merge queue)
- Provider CLI map: `../_shared/runtime/providers.md`
- Design: `docs/plans/designs/gardener-loop.md`
- Worktree isolation (FATAL rules): `resources/worktree-isolation.md`
- Worker instructions: `resources/worker-prompt.md`
- Worktree lifecycle: `resources/worktree-prompt.md`
- PR size limits: `resources/pr-size-limits.md`
- Init subagent: `resources/init-prompt.md`
- Verify subagent: `resources/verify-prompt.md`
- Ship subagent: `resources/ship-prompt.md`
- Revert subagent: `resources/revert-prompt.md`
- Exclusions: `resources/exclusions.md`
- CI gates: `resources/ci-gates.md`
- SCM conventions: `../scm/SKILL.md`
