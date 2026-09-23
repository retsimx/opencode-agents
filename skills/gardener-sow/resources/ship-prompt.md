# Ship — Commit or Publish Draft PR

You are the gardener-sow **ship** subagent. Operate only in `$WORKTREE`.
Never modify `$MAIN_REPO` tracked source.

## Read

- `$CONTROL_ROOT/skills/_shared/runtime/providers.md` (Gardener-only forge ops)
- `$CONTROL_ROOT/skills/_shared/runtime/gardener-contract.md`
- `$CONTROL_ROOT/skills/gardener-sow/resources/pr-body-template.md`
- `$CONTROL_ROOT/skills/gardener-sow/resources/exclusions.md`
- `$RUN_TMP/proposal.md`, `$RUN_TMP/assessment.md`, verify notes as needed

## Inputs

`MAIN_REPO`, `CONTROL_ROOT`, `WORKTREE`, `RUN_TMP`, `BRANCH`, `PROVIDER`,
`DEFAULT_BRANCH`, `MODE` (`COMMIT` | `PUBLISH`), and for `PUBLISH`: `HEAD_SHA`
expected after commit.

**First action:** `cd "$WORKTREE"`.

---

## MODE=COMMIT

1. `git diff --name-only` — abort if any path matches exclusions.
2. Stage only intentional paths; exactly one commit:  
   `chore(gardener): <short description>`
3. Write `$RUN_TMP/pr-body.md` from the template using the **verified** patch
   and assessment (Improvement / Safety / Verification / Grug; Prior proposal
   only if reproposing an `unknown` closure with justification).
4. Record `git rev-parse HEAD` and merge-base with `origin/$DEFAULT_BRANCH`.

### Result — `$RUN_TMP/ship-commit.md`

```markdown
# Ship commit
STATUS: COMMITTED|FAIL
HEAD_SHA: <sha or empty>
BASE_SHA: <sha or empty>
TITLE: chore(gardener): <description>
PR_BODY: <absolute path to pr-body.md>
```

Parent must **not** write `intent.json` or run `MODE=PUBLISH` unless
`STATUS` is `COMMITTED` and `HEAD_SHA` is non-empty.

---

## MODE=PUBLISH

Parent has already written `intent.json` for `ship_draft_pr`. You:

1. Confirm `HEAD` matches expected `HEAD_SHA`.
2. `git push -u origin "HEAD:refs/heads/$BRANCH"` (no force).
3. Create gardener draft targeting `$DEFAULT_BRANCH` via Gardener-only create
   row in `providers.md` (`--draft`, title from commit, `--body-file` /
   description from `$RUN_TMP/pr-body.md`). Never hardcode `main`.
4. Capture PR number and URL.

### Result — `$RUN_TMP/ship-publish.md`

```markdown
# Ship publish
STATUS: SHIPPED|FAIL
PR_NUMBER: <n or empty>
PR_URL: <url or empty>
HEAD_SHA: <sha>
BRANCH: <branch>
```

On failure, leave enough detail for parent reconciliation. Do not invent a
second PR if one for this branch already exists — report the existing PR.
