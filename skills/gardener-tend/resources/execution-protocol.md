# Gardener Tend — Execution Protocol

Provider-agnostic. All forge commands come from
[`../../_shared/runtime/providers.md`](../../_shared/runtime/providers.md).
Resolve `PROVIDER` once at Entry and use only that CLI for the run.

## Entry Gate

1. Detect `PROVIDER` from `git remote get-url origin`.
2. Confirm CLI auth (`gh auth status` or `glab auth status`).
3. Fetch all open PRs via List open PRs; normalize fields (`number`, `title`, `createdAt`, `headRefName`, `baseRefName`, `mergeableState`, `isDraft`, `url`).
4. Filter to `chore(gardener)` prefix in title.
5. Sort by `createdAt` ascending (oldest first).
6. On GitLab, resolve and cache `:pid` once.

## Priority-Ordered Checks (per PR)

### 1. CHECK_MERGE

Use View single PR and normalize `mergeableState` per the Mergeability table in `providers.md`.

| Normalized situation | Action |
|----------------------|--------|
| Conflicts (`dirty` / cannot merge) | Rebase in worktree |
| Behind base | Rebase in worktree |
| CI / merge check still running | Skip to next PR |
| Clean | Proceed to next check |

**Rebase procedure:**
1. Create worktree: `git worktree add /tmp/wt-<number> origin/<headRefName>`
2. Fetch latest base: `git fetch origin <baseRefName>`
3. Rebase: `cd /tmp/wt-<number> && git rebase origin/<baseRefName>`
4. Resolve conflicts, stage, continue rebase
5. Run local CI (ruff + tests + coverage)
6. Push: `git push origin HEAD:refs/heads/<headRefName> --force-with-lease`
7. Clean up: `git worktree remove /tmp/wt-<number>`
8. **Exit skill** (one change per invocation)

### 2. CHECK_CI

Fetch CI / pipeline status for `PROVIDER`.

- Identify ALL failed steps / jobs (not just first)
- If any failures exist → diagnose and fix all in one pass
- Treat `running` / `pending` as skip-to-next (do not fix mid-run)

**CI fix procedure:**
1. Create worktree: `git worktree add /tmp/wt-<number> origin/<headRefName>`
2. For each failed step, read CI logs and diagnose root cause
3. Fix all failures in worktree
4. Run local CI (ruff + tests + coverage)
5. Commit fixes
6. Push: `git push origin HEAD:refs/heads/<headRefName> --force-with-lease`
7. Clean up: `git worktree remove /tmp/wt-<number>`
8. **Exit skill**

### 3. CHECK_COMMENTS

Fetch inline review comments and conversation comments/notes per `providers.md`.

Filter by specific user login when configured (e.g. `retsimx`).

| Comment type | Action |
|-------------|--------|
| Contains code snippet | Implement the suggested change |
| Vague / not actionable | Reply asking for clarification, **do not delete** |
| "Rebase" | Already handled by CHECK_MERGE — skip |

**Comment fix procedure:**
1. Create worktree: `git worktree add /tmp/wt-<number> origin/<headRefName>`
2. Implement the change
3. Run local CI (ruff + tests + coverage)
4. Commit fix
5. Push: `git push origin HEAD:refs/heads/<headRefName> --force-with-lease`
6. Delete the addressed comment/note via Delete a comment/note for `PROVIDER`
7. Clean up: `git worktree remove /tmp/wt-<number>`
8. **Exit skill**

Vague comments: reply with Reply asking clarification for `PROVIDER`, then exit without deleting.

### 4. CHECK_DRAFT

Only if PR is draft (`isDraft: true`):
- Check that all CI checks/pipelines are complete AND passing
- Check there are no unresolved comments
- If healthy → Promote draft → ready for `PROVIDER`
- **Exit skill**

### 5. ALL_CLEAR

If no PR required any action → no-op. Exit skill.

## Verification

Before every push, run:

```bash
poetry run ruff check .
poetry run ruff format --check .
poetry run bash test_coverage.sh
```

**All must pass.** Never push failing CI. Also respect Local CI config discovery for `PROVIDER` when project CI differs.

## Comment Deletion

- Only delete a comment after the fix is pushed.
- Confirm the push succeeded before deleting.
- If multiple comments on the same PR, delete only the one that was addressed.

## Error Recovery

| Error | Action |
|-------|--------|
| Forge CLI not authenticated | Exit with error |
| Worktree create fails | Skip PR, try next |
| Rebase conflict cannot be resolved automatically | Skip PR, report |
| Push rejected (remote changes) | Fetch latest, retry |
| Local verification fails | Fix the issue (verification cannot fail by design) |
| API rate limited | Wait and retry |
