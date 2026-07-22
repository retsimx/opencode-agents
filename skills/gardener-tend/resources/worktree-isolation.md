# Gardener Tend — Worktree Isolation (FATAL)

**Any modification to the main repository checkout is a FATAL ERROR.**

## Definitions

- `MAIN_REPO` — the primary git checkout (where `.git` lives and `git worktree add` is run from)
- `WORKTREE` — the iteration-specific path, e.g. `/tmp/wt-<pr-number>`

## Rules

1. Every subagent MUST receive `WORKTREE` (absolute path) when a worktree exists.
2. Every subagent MUST `cd "$WORKTREE"` before any file read, edit, write, or command.
3. Subagents MUST NOT edit, stage, commit, or run tests in `MAIN_REPO`.
4. `git worktree add` from `MAIN_REPO` does NOT require a clean main checkout — a dirty main is allowed.
5. WORKTREE_SETUP and WORKTREE_CLEANUP run from `MAIN_REPO` only for `git worktree` commands — they MUST NOT modify tracked files in `MAIN_REPO`.

## Fatal error

If any subagent detects it modified files outside `WORKTREE`, or ran destructive git operations on `MAIN_REPO` (checkout, commit, reset, clean on tracked files), return immediately:

`FATAL|main_modified|<detail>`

## Allowed in MAIN_REPO (SETUP/CLEANUP only)

- `git fetch origin main`
- `git worktree add`, `git worktree remove`, `git worktree prune`
- Reading PR status via forge CLI (`gh` / `glab` per `.agents/skills/_shared/runtime/providers.md`)

## Prohibited in MAIN_REPO

- `git checkout -- .`, `git clean`, `git commit`, `git push`
- `edit` / `write` on any source file under `MAIN_REPO`
- Running tests, ruff, or coverage in `MAIN_REPO`
