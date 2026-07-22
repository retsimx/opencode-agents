# Gardener SHIP Subagent

Commit, push, and open a draft PR for the current iteration.

## Read first

- `.agents/skills/_shared/runtime/providers.md` — **required**; detect provider and use only mapped Create draft PR command
- `.agents/skills/gardener-sow/resources/worktree-isolation.md` — **FATAL if violated**
- `AGENTS.md`
- `.agents/skills/scm/SKILL.md`
- `.agents/skills/gardener-sow/resources/exclusions.md`
- `.agents/skills/gardener-sow/resources/pr-size-limits.md`
- `docs/plans/designs/gardener-pr-history.md`

## Inputs (from orchestrator) — REQUIRED

- `WORKTREE` — absolute path to the iteration worktree (**mandatory**)
- `MAIN_REPO` — absolute path to main repository root (do not modify)
- `ITERATION` — zero-padded number, e.g. `0042`
- `SLUG` — kebab-case from SCAN
- `DESCRIPTION` — one-line summary
- `BRANCH` — branch name created at SETUP
- `PROVIDER` — `github` or `gitlab` (from INIT; re-detect from `origin` if missing)

**First action: `cd "$WORKTREE"`.** All git/forge CLI commands run inside `WORKTREE` only.

## Steps

All commands MUST be wrapped with `timeout 300`.

1. Verify `timeout 300 git diff --stat` is within `pr-size-limits.md`.
2. Stage only iteration files (explicit paths).
3. Commit: `timeout 300 git commit -m "chore(gardener): <description>"` — exactly one commit.
4. `timeout 300 git push -u origin HEAD`
5. Create a draft PR with the Create draft PR command for `PROVIDER` in `.agents/skills/_shared/runtime/providers.md`:
   - title: `chore(gardener): <description>`
   - base/target: `main`
   - body: per `pr-size-limits.md`
6. Append to `$MAIN_REPO/.agents/results/gardener-state.md`: `| <ITERATION> | PASS | <SLUG> | <DESCRIPTION> | <BRANCH> | <PR_URL> | open | <ISO_TIMESTAMP> |`

## Return

Exactly one line:

- `SHIPPED|<branch>|<pr_url>`
- `SHIP_FAIL|<reason>`
- `FATAL|main_modified|<detail>`
