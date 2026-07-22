---
name: gardener
description: >
  Autonomous infinite software-gardening loop. Each iteration finds one micro-improvement
  in an isolated git worktree, verifies CI locally, and opens a small human-reviewable
  draft PR. No user interaction. Orchestration-only — all work runs in task subagents.
---

# Gardener — Autonomous Software Gardener

## Scheduling

### Goal
Improve the repository by one micro-change per iteration, forever. Each iteration runs in its own git worktree, produces one small human-reviewable draft PR, then cleans up the worktree. The orchestrator runs only the loop; every substantive action is delegated to task subagents.

### Intent signature
- User invokes `/gardener` or asks to start the software gardener loop
- User wants endless autonomous repo improvement
- User wants one small draft PR per iteration with full CI verification before push

### When to use
- Continuous autonomous code quality improvement
- Relentless micro-optimization using local model economics
- Background gardening with draft PRs for human review

### When NOT to use
- Bounded task with defined completion criteria -> use `ralph`
- Review-only loop until clean -> use `ralphreview`
- Single issue with user approval gates -> use `issue-autopilot`
- One-shot review -> use `deep-review`

### Expected inputs
- Repository with `gh` (GitHub) or `glab` (GitLab) CLI authenticated for `origin`
- `AGENTS.md` and `TESTING.md` present
- Optional: existing `.agents/results/gardener-state.md` from prior runs
- Main checkout may be dirty — gardener does not require or enforce a clean `main`

### Expected outputs
- One small draft PR per successful iteration (`gardener/iter-{NNNN}-{slug}`)
- Log in `.agents/results/gardener-state.md` with Status and Description columns; rows removed once PR is merged
- No orphaned worktrees after any iteration
- Main checkout state unchanged by gardener
- Infinite loop until externally killed

### Dependencies
- OpenCode `task` tool — all scan, work, verify, ship, worktree, and revert work
- `scm` skill — loaded by SHIP subagent for commit conventions
- Shared provider map: `../_shared/runtime/providers.md` (`gh` / `glab`)
- Resources: `resources/worktree-isolation.md`, `resources/worker-prompt.md`, `resources/worktree-prompt.md`, `resources/pr-size-limits.md`, `resources/exclusions.md`, `resources/ci-gates.md`, `resources/init-prompt.md`, `resources/verify-prompt.md`, `resources/ship-prompt.md`, `resources/revert-prompt.md`
- Forge CLI (`gh` or `glab`) for draft PR creation
- Design doc: `docs/plans/designs/gardener-loop.md`

### Control-flow features
- Infinite loop with no max iterations and no safeguard exit
- Each iteration uses an isolated git worktree; main checkout never modified
- Mandatory worktree cleanup after every iteration (success or failure)
- Every subagent receives `MAIN_REPO` and `WORKTREE` paths
- `FATAL|main_modified` aborts iteration immediately
- No `question` tool — fully autonomous

## Structural Flow

### Entry
1. Verify this is an autonomous gardening session (no user prompts).
2. Record `MAIN_REPO` as absolute path to main repository checkout.
3. Initialize or read `.agents/results/gardener-state.md`.
4. Enter the infinite loop at INIT.

### Scenes
1. **INIT**: Task subagent (`init-prompt.md`) detects provider, verifies forge CLI auth, fetches `origin main`, reads state; does not check main cleanliness; returns `READY|<iteration>`.
2. **WORKTREE_SETUP**: Task subagent (`worktree-prompt.md`, MODE=SETUP) creates isolated worktree from `origin/main` (works even if main is dirty); returns `WORKTREE_READY|<path>|<branch>`.
3. **SCAN**: Task subagent (`worker-prompt.md`, MODE=SCAN) inside `WORKTREE`; returns `FOUND|...`, `SKIP|...`, or `FATAL|main_modified|...`.
4. **WORK**: Task subagent (`worker-prompt.md`, MODE=WORK) inside `WORKTREE`; returns `SUCCESS|...`, `FAIL|...`, `ABORT|...`, or `FATAL|main_modified|...`.
5. **VERIFY**: Task subagent (`verify-prompt.md`) inside `WORKTREE`; returns `PASS`, `FAIL:<reason>`, or `FATAL|main_modified|...`.
6. **SHIP**: Task subagent (`ship-prompt.md`) inside `WORKTREE`; returns `SHIPPED|<branch>|<pr_url>`, `SHIP_FAIL:<reason>`, or `FATAL|main_modified|...`.
7. **LOG**: SHIP subagent appends iteration row with Status=open; INIT removes rows whose PRs are merged, updates rows whose PRs are closed (Status=open → Status=closed), and removes rows with Status=done.
8. **WORKTREE_CLEANUP**: Task subagent (`worktree-prompt.md`, MODE=CLEANUP) removes worktree — **mandatory, always runs**; does not alter main working tree.
9. **LOOP**: Increment iteration, return to INIT.

### Transitions
- WORKTREE_SETUP fails -> LOG, loop (no cleanup needed).
- Any `FATAL|main_modified` -> WORKTREE_CLEANUP, LOG `FATAL`, loop.
- SCAN returns `SKIP` -> WORKTREE_CLEANUP, LOG `SKIP`, loop.
- WORK returns `FAIL` or `ABORT` -> revert via task, WORKTREE_CLEANUP, LOG, loop.
- VERIFY returns `FAIL` -> revert via task, WORKTREE_CLEANUP, LOG, loop.
- SHIP returns `SHIP_FAIL` -> revert via task, WORKTREE_CLEANUP, LOG, loop.
- SHIP returns `SHIPPED` -> LOG `PASS`, WORKTREE_CLEANUP, loop.
- Any error -> revert if needed, WORKTREE_CLEANUP (always), LOG, loop.

### Failure and recovery
| Failure | Recovery |
|---------|----------|
| `FATAL\|main_modified` | cleanup worktree, log `FATAL`, loop |
| VERIFY fails | revert via task, cleanup worktree, log `FAIL:<reason>`, loop |
| SHIP fails | revert via task, cleanup worktree, log `SHIP_FAIL`, loop |
| SCAN finds nothing | cleanup worktree, log `SKIP`, loop |
| Subagent timeout/error | revert if dirty in worktree, cleanup worktree, log `ERROR:<message>`, loop |
| CLEANUP fails | log `CLEANUP_FAIL`, loop anyway |

### Exit
- No normal exit. Loop runs until the process is killed externally.
- Each iteration is independent; failures never stop the loop.

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
| Loop | `UPDATE_STATE` | increment iteration |

### Tools and instruments
- OpenCode `task` tool (`subagent_type="general"`) — **only** tool for substantive work
- Orchestrator may use `bash` **only** for: appending to state file, reading iteration counter
- Orchestrator **MUST NOT**: `read` source files, `edit`/`write` source files, run ruff/tests/coverage, run `git commit`/`git push`/forge PR create, create/remove worktrees

### Canonical workflow path

**CRITICAL: The orchestrator is a thin loop. It MUST NOT perform scan, work, verify, ship, or worktree operations inline. Every phase except LOG uses the built-in `task` tool. Never run `$ task` in bash.**

**Every task subagent prompt MUST include `MAIN_REPO` (absolute path) and `WORKTREE` (absolute path, once created). Nested subagents inherit both. See `resources/worktree-isolation.md`.**

**All git/gh/bash commands inside subagents MUST be wrapped with `timeout 300` to prevent infinite hangs.**

```
STATE_FILE = .agents/results/gardener-state.md
MAIN_REPO = absolute path to main repository checkout
iteration = read from STATE_FILE or 0

while true:

    worktree = null
    branch = null

    # INIT — resources/init-prompt.md
    Call task (gardener-init) with MAIN_REPO
    Return: READY|<iteration_number> or NOT_READY|<reason>
    if NOT_READY: log, continue

    # WORKTREE_SETUP — resources/worktree-prompt.md MODE=SETUP, SLUG=pending
    Call task (gardener-worktree-setup) with MAIN_REPO, ITERATION, SLUG=pending
    Return: WORKTREE_READY|<worktree_path>|<branch> or WORKTREE_FAIL|<reason>
    if WORKTREE_FAIL: log, iteration++, continue

    worktree = path; branch = branch name

    # SCAN — resources/worker-prompt.md MODE=SCAN
    Call task (gardener-scan) with MAIN_REPO, WORKTREE
    Return: FOUND|<slug>|... or SKIP|... or FATAL|main_modified|...

    if FATAL: Call task (gardener-worktree-cleanup) with MAIN_REPO, WORKTREE, BRANCH; log; iteration++; continue
    if SKIP: Call task (gardener-worktree-cleanup) with MAIN_REPO, WORKTREE, BRANCH; log; iteration++; continue

    # WORK
    Call task (gardener-work) with MAIN_REPO, WORKTREE, ITEM=<slug>|<description>|<rationale>
    Return: SUCCESS|... or FAIL|... or ABORT|... or FATAL|main_modified|...

    if FATAL/FAIL/ABORT:
        Call task (gardener-revert) with MAIN_REPO, WORKTREE
        Call task (gardener-worktree-cleanup) with MAIN_REPO, WORKTREE, BRANCH
        log, iteration++, continue

    # VERIFY
    Call task (gardener-verify) with MAIN_REPO, WORKTREE
    Return: PASS or FAIL:<reason> or EXCLUDED:<path> or FATAL|main_modified|...

    if not PASS or FATAL:
        Call task (gardener-revert) with MAIN_REPO, WORKTREE
        Call task (gardener-worktree-cleanup) with MAIN_REPO, WORKTREE, BRANCH
        log, iteration++, continue

    # SHIP
    Call task (gardener-ship) with MAIN_REPO, WORKTREE, BRANCH, ITERATION, SLUG, DESCRIPTION
    Return: SHIPPED|... or SHIP_FAIL|... or FATAL|main_modified|...

    if SHIP_FAIL or FATAL:
        Call task (gardener-revert) with MAIN_REPO, WORKTREE
        Call task (gardener-worktree-cleanup) with MAIN_REPO, WORKTREE, BRANCH
        log, iteration++, continue

    # LOG — SHIP already appended the state row above (Status=open); INIT removes merged and done rows, updates closed rows

    # WORKTREE_CLEANUP — mandatory after every iteration
    Call task (gardener-worktree-cleanup) with MAIN_REPO, WORKTREE, BRANCH
    iteration++
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
- One new branch and one draft PR per successful iteration
- Worktree removed after every iteration; main checkout state preserved
- Never commits to `main`

### Guardrails
1. **Orchestrator never edits source** — no `read`/`edit`/`write` on application code
2. **Orchestrator never runs CI** — ruff, tests, coverage run only in subagents
3. **No `question` tool** — fully autonomous, no user interaction
4. **No stop conditions** — loop forever until externally killed
5. **One fix per iteration** — one commit, one draft PR, one specific concern
6. **Never commit to or modify `main`** — `FATAL|main_modified` on violation
7. **Isolated worktree per iteration** — all SCAN/WORK/VERIFY/SHIP inside `WORKTREE` only
8. **Every subagent gets `MAIN_REPO` + `WORKTREE`** — nested subagents inherit both
9. **Dirty main is allowed** — worktree created from `origin/main` without cleaning main
10. **Mandatory cleanup** — worktree removed after every iteration regardless of outcome
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
