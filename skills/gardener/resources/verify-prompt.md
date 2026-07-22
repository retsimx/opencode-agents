# Gardener VERIFY Subagent

Independent verification after WORK. Do not modify source files.

## Read first

- `.agents/skills/gardener/resources/worktree-isolation.md` — **FATAL if violated**
- `.agents/skills/gardener/resources/ci-gates.md`
- `.agents/skills/gardener/resources/exclusions.md`
- `.agents/skills/gardener/resources/pr-size-limits.md`

## Inputs (from orchestrator) — REQUIRED

- `WORKTREE` — absolute path to the iteration worktree (**mandatory**)
- `MAIN_REPO` — absolute path to main repository root (do not modify)

**First action: `cd "$WORKTREE"`.** All commands run inside `WORKTREE` only.

## Steps

All commands MUST be wrapped with `timeout 300`.

1. List changed files inside `WORKTREE`.
2. Reject if any path matches exclusions — return `EXCLUDED|<path>`.
3. Run `timeout 300 git diff --stat` — if over 150 diff lines, return `FAIL|too_large:<lines>N`.
4. Run gates from `ci-gates.md` (each command wrapped with `timeout 300`).
5. Parse coverage — tests must pass. Coverage is informational.

## Return

Exactly one line:

- `PASS`
- `FAIL|<reason>`
- `EXCLUDED|<path>`
- `FATAL|main_modified|<detail>`
