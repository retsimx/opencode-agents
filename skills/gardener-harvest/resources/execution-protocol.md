# Gardener Harvest — Execution Protocol

Normative runtime detail for `gardener-harvest`. Load with
`gardener-contract.md`, `gardener-state.md`, and shared `providers.md`.

Forge commands: only
`.agents/skills/_shared/runtime/providers.md` (stub:
`.agents/skills/gardener-harvest/resources/providers.md`).

## Paths

| Name | Value |
|------|--------|
| `CONTROL_ROOT` | Walk parents of this skill until `skills/gardener-sow` exists |
| `MAIN_REPO` | Target git repository (cwd for forge + git) |
| `RUN_DIR` | `$CONTROL_ROOT/results/tmp/pr-harvest/<run-id>/` |
| State | `$CONTROL_ROOT/state/gardener/{open,closed,intent}.json` |

`run-id` must be unique per incomplete capture (UTC timestamp + optional suffix).

## Entry cleanup

1. If intent `skill` is another gardener skill → stop; report it.
2. Worktree cleanup (same pin as sow/tend): remove
   `$CONTROL_ROOT/worktrees/gardener-*` paths that are **not**
   `intent.worktree_path`. No intent / empty path → remove all such paths.
3. Delete every directory under `$CONTROL_ROOT/results/tmp/pr-harvest/` whose
   path is **not** equal to `intent.run_dir`. No intent / empty `run_dir` →
   delete all of them.
4. Validate `closed.json` (stop if present but invalid). Rebuild `open.json`
   if absent/invalid/wrong repo.
5. If `skill == "harvest"` and `intent.json` exists → reconcile (below), then
   **exit**. Do not SELECT.

## Intent operations

Allowed harvest operations only:

- `harvest_merge`
- `harvest_needs_fix`
- `harvest_close`

Write intent **immediately before** the forge effect. Fields at minimum:

```json
{
  "skill": "harvest",
  "repository": "provider-host/owner/project",
  "operation": "harvest_merge",
  "subject": "42",
  "expected_result": "PR 42 MERGED; source branch delete requested",
  "worktree_path": "",
  "run_dir": "/abs/path/to/pr-harvest/<run-id>",
  "run_id": "<run-id>",
  "pr_number": 42,
  "head_sha": "<assessed head>",
  "base_sha": "<assessed base>"
}
```

Delete intent only after the **whole** expected result is verified.

### Reconcile on entry

Query forge/git before retrying.

**harvest_merge**

- Already merged → verify requested source-branch deletion (404 OK; if still
  present report `merged_branch_not_deleted` — never delete manually), remove
  open-cache entry, clear intent.
- Still open and recorded base/head SHAs still match → retry pinned expected-head
  merge once.
- Still open and either SHA changed → verify no merge occurred, clear stale
  intent, treat as `SKIP` for this invocation, **exit** (do not SELECT).

**harvest_needs_fix**

- Recorded SHAs still match → query comments for `<!-- gardener -->` and
  equivalent fix text; add comment only if absent; then clear intent when
  comment present.
- SHA changed → clear stale no-effect intent; `SKIP`; **exit**.

**harvest_close**

- Recorded SHAs still match **or** PR already closed → ensure rationale comment
  with marker exists, ensure closed, write `closed.json` category
  (human non-marked reject/close → `rejected` + `suppress_equivalent: true`;
  else `unknown` + `suppress_equivalent: false`), remove open entry, clear
  intent.
- Still open and SHA changed → clear stale intent; `SKIP`; **exit**.

## SELECT — one PR

1. Fully paginate open PRs (providers list rules).
2. Normalize fields client-side.
3. Keep only PRs where **all** hold:
   - title starts with `chore(gardener):`
   - head branch starts with `gardener/run-` (never `gardener/iter-`)
   - `baseRefName` == resolved default branch
   - same-repository head (not a fork)
   - `isDraft == false`
4. Sort by `createdAt` ascending.
5. Process **only index 0**. Exit after that PR's path completes.

Do not maintain a durable "processed" list for temporary CI/conflict skips.

## EARLY_SKIP (no assessor)

From list/view + CI metadata **before** capture:

| Condition | Action |
|-----------|--------|
| Conflict / unmergeable / dirty merge state | Update open observation; `SKIP`; exit |
| Any check pending or running | Update open observation; `SKIP`; exit |
| Any check failed | Update open observation; `SKIP`; exit |

Pending vs failed vs green comes from `gh pr checks` / pipeline jobs, **not**
from the shared providers mergeability row that maps `UNSTABLE` to
“CI running — skip.”

"Update open observation" means refresh that item's `head_sha` / metadata in
`open.json` when known. Do not invent `topic`/`rationale`/`areas` from the title.

Zero checks observed is **not** an early skip by itself — continue to capture
and let assessor + parent judge; MERGE still requires green CI at recheck
(absence of checks is not "green").

## CAPTURE — `RUN_DIR` layout

Create `RUN_DIR` and write at least:

| File | Content |
|------|---------|
| `meta.json` | number, url, title, branch, base/head SHAs, default branch, provider |
| `description.md` | PR body |
| `patch.diff` | full PR diff |
| `ci.txt` | CI / pipeline summary |
| `discussions.md` | review + conversation comments |
| `subagent-ledger.json` | assessor task ledger (after spawn) |
| `assessment.md` | assessor output (after assess) |

Pass absolute `MAIN_REPO`, `CONTROL_ROOT`, `RUN_DIR`, `PROVIDER`, PR number,
and pinned base/head SHAs to the assessor.

## Assessor dispatch gate

Ledger (`$RUN_DIR/subagent-ledger.json`):

```json
{
  "subagents": {
    "assessor": {
      "role": "assessor",
      "task_id": "<harness-returned-id>",
      "result_file": "<abs>/assessment.md",
      "status": "complete"
    }
  }
}
```

Before PARENT_DECIDE, require all of:

- non-empty `task_id`
- `status == complete`
- `assessment.md` exists and is non-empty
- required section markers present (see assessor-prompt):
  `# Verdict`, `# Valuable micro-improvement`, `# Intended behavior`,
  `# Behavior change`, `# Coherence`, `# Unhinged or nonsense`,
  `# Correctness`, `# Tests`, `# Grug`, `# PR description claims`,
  `# Justification`

Inline parent notes may supplement; they never replace `assessment.md`.

On gate failure: re-dispatch the assessor once. If still failing → `SKIP`,
retain `RUN_DIR`, stop without forge mutation.

## SHA recapture

Immediately before MERGE / NEEDS_FIX / CLOSE:

1. Re-fetch base SHA, head SHA, mergeability, CI.
2. If base or head differs from captured assessment SHAs:
   - Discard prior assessment
   - Recapture into the same or a fresh `RUN_DIR`
   - Spawn a new assessor; pass dispatch gate; parent re-decides
   - This may happen **at most once** per invocation
3. If after that one recapture either SHA still mismatches → `SKIP`; exit

## MERGE procedure

Preconditions at mutate time: child+parent MERGE, CI green, mergeability clean,
no unresolved actionable review, head SHA == assessed head.

1. Write `harvest_merge` intent (`run_dir`, SHAs, expected merged state).
2. Issue pinned expected-head squash-merge (providers Gardener-only table).
3. Verify PR state merged.
4. Verify branch deletion (404); if not deleted, record failure — do not delete
   manually.
5. Remove open-cache entry (atomic write).
6. Clear intent; delete `RUN_DIR`.

If squash/expected-head is rejected → do not fall back; `SKIP` with
`merge_strategy_mismatch` (or equivalent). Clear or retain intent per whether
any effect occurred (query first).

## NEEDS_FIX procedure

1. Search existing comments for `<!-- gardener -->` and near-equivalent ask.
2. If equivalent exists → do not comment again; clear any stale intent; exit.
3. Write `harvest_needs_fix` intent.
4. Post comment whose body includes the exact line `<!-- gardener -->` plus a
   concrete actionable request.
5. Verify comment present; clear intent; delete `RUN_DIR`.

## CLOSE procedure

1. Determine rationale source:
   - Explicit **non-marked** human reject/close → authoritative; category
     `rejected`
   - Else child+parent CLOSE agreement → category `unknown`
2. Write `harvest_close` intent.
3. Post closure rationale with `<!-- gardener -->` (agent CLOSE comments are
   marked and must **not** set suppress).
4. Close the PR.
5. Verify closed; append `closed.json` item with correct category /
   `suppress_equivalent`; remove open entry.
6. Clear intent; delete `RUN_DIR`.

## Comment marker

Every gardener-authored harvest comment must include this exact line:

```
<!-- gardener -->
```

`rejected` / suppress is allowed only when a **non-marked** comment explicitly
asks to reject or close.

## Open / closed writes

Follow gardener-state atomic write rules. Never overwrite invalid `closed.json`.
Categories used by harvest: `rejected` | `unknown` only.

## Forbidden

- Risk / confidence / ASK / size gates / `repo-rules.yaml`
- Waiting or polling for CI
- Rebase, push, edit, CI repair, manual branch delete
- Processing more than one PR per invocation
- Recognizing `gardener/iter-`
- Merging without expected-head pin when the pin cannot be issued
- Durable suppression of temporary CI / conflict skips
