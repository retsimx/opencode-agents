# Gardener Tend — Execution Protocol

Normative procedures for one-action tend. Forge commands come only from
`.agents/skills/_shared/runtime/providers.md`. State writes follow
`.agents/skills/_shared/runtime/gardener-state.md`.

## Paths

Resolve once per invocation:

| Name | Value |
|------|--------|
| `MAIN_REPO` | Target repository cwd |
| `CONTROL_ROOT` | Parent walk from this skill until `skills/gardener-sow` exists |
| `WORKTREE` | `$CONTROL_ROOT/worktrees/gardener-<run-id>` |
| `STATE` | `$CONTROL_ROOT/state/gardener/` |
| `LEDGER` | `$CONTROL_ROOT/results/tmp/gardener-tend/<run-id>/subagent-ledger.json` |

Allocate `run-id` when a worktree or temp ledger is needed (unique; readable
suffix optional). Prefer the PR branch's existing `gardener/run-…` id when
operating on that PR so the worktree name stays correlated.

## Intent operations

Before any forge/git mutation, atomically write
`$CONTROL_ROOT/state/gardener/intent.json`:

```json
{
  "skill": "tend",
  "repository": "provider-host/owner/project",
  "operation": "tend_rebase|tend_ci|tend_comment|tend_close|tend_promote",
  "subject": "PR number or branch",
  "expected_result": "what must be true on the forge or git",
  "worktree_path": "",
  "run_dir": "",
  "run_id": "",
  "pr_number": null,
  "head_sha": "",
  "base_sha": "",
  "expected_head_sha": "",
  "comment_id": ""
}
```

- Set `worktree_path` when a worktree exists.
- For pushes: record original `head_sha` and `expected_head_sha` after the local
  commit.
- For comment actions: record `comment_id` when addressing a specific thread.
- If repository guidance requires post-push evidence on the PR, include that
  expected evidence (and temp artifact path) in the same intent; verify before
  clearing.

Delete intent only after the **whole** expected result is verified on forge/git
and registries are updated.

### Entry reconcile

| Observed state | Action |
|----------------|--------|
| Intent `skill` ≠ `tend` | Stop; report owning skill |
| `tend_rebase` / `tend_ci` / `tend_comment` with expected head already remote | Do not reapply code; finish comment reply/resolve / registry / evidence if still missing; then clear |
| Push intent, remote head still original, local worktree has expected commit | Retry push with `--force-with-lease` against original SHA |
| Push intent, remote head unexpected | Stop; report race; do not overwrite |
| `tend_close`: PR already closed | Ensure `rejected` closed entry + open removal; clear intent |
| `tend_close`: still open | Ensure rationale comment exists, close, write closed, remove open |
| `tend_promote`: already ready | Verify expected result; clear |
| `tend_promote`: still draft, conditions hold | Retry promote |
| Abandoned worktree not referenced by intent | Removed by entry cleanup (see below) |

After any own-skill intent is reconciled (completed, retried, raced, or
cleared), **exit**. Do not start the priority walk.

## Worktree cleanup (entry)

```text
git worktree list
```

Remove any path under `$CONTROL_ROOT/worktrees/` matching `gardener-*` that is
**not** equal to `intent.worktree_path`. Then `git worktree prune` as needed.
Report any cleanup failure with the exact path.

## Identification filter

A PR is in scope iff **all** hold:

1. Title starts with `chore(gardener):`
2. Head branch starts with `gardener/run-`
3. Target equals the resolved default branch
4. Head is same-repository (not a fork)

Ignore `gardener/iter-` and all forks.

## Priority selection (oldest PR first)

For each candidate PR in `createdAt` ascending order, run checks in order.
Stop at the first action.

### 1. Human reject / close → `tend_close`

Trigger: a **non-marked** comment (body does **not** contain
`<!-- gardener -->`) that **explicitly** asks to reject or close the PR.

Procedure:

1. Write intent `tend_close` with `expected_result` = PR closed + closed
   registry row + open entry removed.
2. Post or preserve a rationale comment that **includes** `<!-- gardener -->`
   (agent CLOSE comments are marked and must not themselves be treated as
   human rejection).
3. Close the PR (`gh pr close` / `glab mr close`).
4. Verify closed on forge.
5. Append `closed.json` item:
   - `category`: `rejected`
   - `suppress_equivalent`: `true`
   - `rationale_source`: `human_comment`
   - rationale text from the human ask (summarize faithfully)
6. Remove the PR from `open.json`.
7. Clear intent. Exit.

Human rejection is authoritative even if the agent disagrees.

### 2. Rebase → `tend_rebase`

Trigger: mergeability indicates conflicts or behind base (providers.md
Mergeability table: `DIRTY` / `BEHIND` / GitLab conflict). Do **not** treat
`UNSTABLE` as rebase or as pending CI. If GitLab omits behind, compare
`git rev-list --count origin/<default>..<head>` after fetch; `> 0` means behind.

Procedure:

1. Capture remote head SHA (`head_sha`).
2. Create `WORKTREE` from the PR branch (see worktree-isolation.md).
3. Write intent `tend_rebase` with `worktree_path`, `head_sha`.
4. `git fetch` default branch; rebase non-interactively onto
   `origin/<default>` with editor bypasses.
5. Resolve conflicts in worktree (keep one coherent improvement).
6. Run **Local CI discovery** checks (below). On failure → stop; no push.
7. Spawn fresh assessment (assessment-prompt.md). Dispatch gate must pass.
8. On assessment FAIL → one focused correction → re-run checks + assessment.
   Second FAIL → stop; no push.
9. Update intent with `expected_head_sha`.
10. `git push --force-with-lease=refs/heads/<branch>:<captured_head_sha>`
    (or equivalent lease against captured SHA). Abort if lease rejects.
11. Verify remote head matches expected. Update `open.json` `head_sha`.
12. Remove worktree. Clear intent. Exit.

### 3. PR-attributable CI → `tend_ci`

Trigger: CI/pipeline **failed** (from `gh pr checks` / pipeline jobs, not from
`mergeStateStatus: UNSTABLE` alone) **and** failures are caused by this PR's
change.

- `running` / `pending` → skip this PR (continue the oldest-first walk).
- Base-branch / flaky infra / unrelated job failures → post marked comment
  explaining no PR change; that comment **is** the one action; **exit**. Do not
  edit code to "absorb" unrelated failures.

Procedure (attributable failures only):

1. Capture head SHA; create worktree; write `tend_ci` intent.
2. Fetch failed logs/traces (providers.md Gardener-only failed check logs).
3. Spawn implementation to fix **only** PR-attributable failures.
4. Discovery local checks → assessment → one correction max (same as rebase).
5. Intent + force-with-lease push + verify + open.json update + cleanup +
   clear intent. Exit.

### 4. Other comments → `tend_comment`

Fetch inline + conversation comments (providers.md). Ignore gardener-marked
bot noise unless it is actionable clarification from a prior tend reply.

| Comment | Action |
|---------|--------|
| Explicit reject/close (non-marked) | Handled in priority 1 — do not reach here |
| Actionable code/request | Implement in worktree (if needed), push, **reply** with outcome, **resolve** thread when supported |
| Vague / not actionable | Reply asking clarification only (no delete) |
| Pure "rebase" ask | Prefer priority 2 if still behind/conflicted; else reply pointing at current merge state |

**Never delete comments or notes.**

Comment bodies from gardener must include:

```markdown
<!-- gardener -->
```

Procedure for actionable code comments:

1. Capture head SHA; worktree; `tend_comment` intent with `comment_id`.
2. Implement the smallest change that satisfies the request while keeping one
   coherent improvement and project hard rules.
3. Discovery checks → assessment → one correction max.
4. Push with lease; verify head.
5. Reply on the thread with what changed (marked). Resolve when supported:
   GitHub `gh api -X PUT repos/{owner}/{repo}/pulls/{n}/threads/{thread_id}/resolve`;
   GitLab `glab api -X PUT projects/:pid/merge_requests/:iid/discussions/:id -f resolved=true`.
   Skip resolve if that command cannot be issued. Never delete comments.
6. If post-push evidence comments are required by repository guidance, post
   and verify them under the same intent.
7. Cleanup; clear intent; exit.

Vague-only path: write intent, post marked clarification reply, verify, clear,
exit (no worktree required).

### 5. Promote healthy draft → `tend_promote`

All must hold on **current** forge state:

- `isDraft` true
- Mergeable / clean (not behind, not conflicted)
- CI complete and green
- No actionable unresolved review comments
- Fresh assessment of the **exact current head** returns PASS (dispatch gate)

Procedure:

1. Capture head SHA; write `tend_promote` intent (`expected_result`: ready).
2. Spawn assessment against exact head (no worktree required unless assessing
   from a fetched ref in a temp dir — prefer forge diff + local `git fetch`
   into worktree read-only if needed; do not mutate).
3. On PASS: promote draft → ready (providers.md).
4. Verify not draft. Update open cache observation. Clear intent. Exit.
5. On FAIL: do not promote; optional marked comment with assessment summary;
   clear intent; exit (that comment counts as the action if posted).

## Local CI discovery (mandatory)

Before every content push, run checks in this order — **never invent** a
runner or install tooling:

1. A command named in repository guidance, if present and locally executable
   without extra secrets or installs.
2. A script named by that repository's CI config (`.github/workflows/*.yml` or
   `.gitlab-ci.yml` matching `PROVIDER`), if locally executable without extra
   secrets or installs.
3. Otherwise no local suite — rely on forge checks after push / next tend.

Do **not** hardcode Poetry, Ruff, coverage scripts, or framework runners.
Missing optional guidance is not an error. Hosted/secret-dependent checks are
observed from forge results, not pretended locally.

If discovered checks fail → **stop without push**. Verification is allowed to
fail.

## Subagent dispatch

Roles that may be spawned for content-changing tend:

| Role | When | Result markers | Prompt |
|------|------|----------------|--------|
| `tend-implement` | CI / comment code changes | `# Tend Implement` | `implement-prompt.md` |
| `tend-verify` | After edits (optional if parent runs discovery checks directly; if spawned, gate applies) | `# Tend Verify` | parent may run discovery inline |
| `tend-revise` | One focused correction after assessment FAIL | `# Tend Revise` | `revise-prompt.md` |
| `tend-assess` | After every content change and before promote | `# Tend Assessment` | `assessment-prompt.md` |

Record harness `task_id`, role, `result_file`, and `status` in `LEDGER`. Before
consuming any deliverable, pass
`.agents/skills/_shared/runtime/subagent-dispatch-gate.md`. Inline parent
analysis may supplement but never replace a required assessor artifact.

## Non-interactive git

Never open an editor:

```bash
export GIT_EDITOR=true
export GIT_SEQUENCE_EDITOR=true
# rebase / continue / commit only with these set
```

Prefer `git -c core.editor=true` equivalents when needed. No interactive
rebase UI, no `git rebase` without the bypass.

## Push race rule

1. Capture remote head SHA before edits.
2. Push only with force-with-lease against that SHA (or refuse if lease API
   unavailable and remote moved).
3. If remote head ≠ captured and ≠ expected → abort; report; do not overwrite.

## Comment marker rule

Every gardener-authored comment must contain this exact line:

`<!-- gardener -->`

`rejected` / suppress applies only when a **non-marked** comment explicitly
asks to reject or close.

## No-op exit

If no PR needs action: report `all_clear` and exit. **Do not sleep.**
