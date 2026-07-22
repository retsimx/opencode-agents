# Gardener Worktree — SETUP + CLEANUP

Each iteration runs in an isolated git worktree. The main repo checkout must never be modified.

## Read first

- `.agents/skills/gardener/resources/worktree-isolation.md`

## Inputs (from orchestrator)

- `MAIN_REPO` — absolute path to main repository root
- `ITERATION` — zero-padded number, e.g. `0042`
- `SLUG` — kebab-case from SCAN (SETUP only, after SCAN — see note below)
- `MODE` — `SETUP` or `CLEANUP`
- `WORKTREE` — absolute path (required for CLEANUP; returned by SETUP)
- `BRANCH` — branch name (required for CLEANUP; returned by SETUP)

## SETUP mode

All commands MUST be wrapped with `timeout 300`.

1. Run from `MAIN_REPO` only. Do not `cd` into an existing gardener worktree.
2. `timeout 300 git fetch origin main` (from `MAIN_REPO`).
3. Compute:
   - `BRANCH=gardener/iter-{ITERATION}-{SLUG}` (or `gardener/iter-{ITERATION}-pending` if SETUP runs before SCAN — orchestrator may pass `SLUG=pending` and rename is NOT required; branch slug is finalized at SHIP)
   - `WORKTREE=<repo-parent>/sarahwebsite-gardener-{ITERATION}`
4. If `WORKTREE` path exists: `timeout 300 git worktree remove --force <WORKTREE>` or `rm -rf` + `timeout 300 git worktree prune`.
5. Create worktree from `origin/main` — **works even if `MAIN_REPO` is dirty**:
   ```
   timeout 300 git -C <MAIN_REPO> worktree add -b <BRANCH> <WORKTREE> origin/main
   ```
6. Do NOT run `git checkout`, `git pull`, `git reset`, or modify any files in `MAIN_REPO`.

## Return (SETUP)

Exactly one line:

- `WORKTREE_READY|<absolute_worktree_path>|<branch>`
- `WORKTREE_FAIL|<reason>`

## CLEANUP mode

**Mandatory after every iteration — success, failure, SKIP, FATAL, or error.**

All commands MUST be wrapped with `timeout 300`.

1. `cd` to `MAIN_REPO` (for git worktree commands only).
2. If `WORKTREE` exists: `timeout 300 git worktree remove --force <WORKTREE>`.
3. `timeout 300 git worktree prune`.
4. If branch was never pushed: `timeout 300 git branch -D <BRANCH>`.
5. If branch was pushed (SHIP succeeded): remove worktree only; remote branch stays for the draft PR.
6. **Do NOT** run `git checkout -- .`, `git clean`, or any command that alters `MAIN_REPO` working tree state.

## Return (CLEANUP)

Exactly one line:

- `CLEANED`
- `CLEANUP_FAIL|<reason>` — orchestrator logs and loops anyway

## Rules

- Never run SCAN/WORK/VERIFY/SHIP in `MAIN_REPO`
- Never leave orphaned worktrees under `<repo-parent>/sarahwebsite-gardener-*`
- CLEANUP runs even on `FATAL|main_modified`
- Pass `WORKTREE` to every nested subagent
