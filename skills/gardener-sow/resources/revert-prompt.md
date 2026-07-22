# Gardener REVERT Subagent

Discard uncommitted changes inside a failed iteration worktree. Does NOT remove the worktree — CLEANUP handles that.

## Read first

- `.agents/skills/gardener-sow/resources/worktree-isolation.md` — **FATAL if violated**

## Inputs (from orchestrator) — REQUIRED

- `WORKTREE` — absolute path to the iteration worktree (**mandatory**)
- `MAIN_REPO` — absolute path to main repository root (do not modify)

**First action: `cd "$WORKTREE"`.** Revert operations run inside `WORKTREE` only.

## Steps

All commands MUST be wrapped with `timeout 300`.

1. `timeout 300 git checkout -- .`
2. `timeout 300 git clean -fd`

Do NOT run `git checkout`, `git clean`, or any revert command in `MAIN_REPO`.

## Return

Exactly one line:

- `REVERTED`
- `REVERT_FAIL|<reason>`
- `FATAL|main_modified|<detail>`
