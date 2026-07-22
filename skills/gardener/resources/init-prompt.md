# Gardener INIT Subagent

Verify prerequisites and return readiness. Operate from main repo root only for git/forge CLI checks — do not edit source files.

## Read first

- `.agents/skills/_shared/runtime/providers.md` — **required**; detect provider and use only mapped commands
- `.agents/skills/gardener/resources/worktree-isolation.md`
- `.agents/skills/gardener/resources/../docs/plans/designs/gardener-pr-history.md`

## Inputs (from orchestrator)

- `MAIN_REPO` — absolute path to main repository root

## Steps

All commands MUST be wrapped with `timeout 300`.

1. From `MAIN_REPO`: detect `PROVIDER` from `git remote get-url origin` per `providers.md`.
2. Run the Auth check for `PROVIDER` (`gh auth status` or `glab auth status`) — abort with `NOT_READY|<cli> not authenticated` if it fails.
3. From `MAIN_REPO`: `timeout 300 git fetch origin main`.
4. Read `.agents/results/gardener-state.md`. Use `edit` with `replaceAll=true` to remove every row where `| done |` appears in the Status column.
5. Re-read the file. Collect all rows where Status=open or Status=closed.
6. For each row where Status=open, extract the PR URL (column 6). Run the View by URL command for `PROVIDER` (normalize `state` per `providers.md`) for each (batch up to 10 at a time). If a PR's normalized state is `merged`/`MERGED`, use `edit` to remove that row from the file. If normalized state is `closed`/`CLOSED`, use `edit` to change the row's Status from `open` to `closed`.
7. Prune stale gardener worktrees: `timeout 300 git worktree prune`.

**Do NOT check whether main is dirty.** A dirty main checkout is allowed. Gardener never modifies main.

## Return

Exactly one line:

- `READY|<iteration_number>` — next iteration number (last + 1, or 1 if fresh). Include provider in logs only; keep the return line format unchanged.
- `NOT_READY|<reason>` — only for forge auth failure or unrecoverable git errors (not dirty main)
