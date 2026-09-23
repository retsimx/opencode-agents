---
name: gardener-harvest
description: >
  Assess and act on exactly one oldest non-draft gardener PR per invocation
  (MERGE, NEEDS_FIX, CLOSE, or SKIP), then exit. Evidence-based child assessor
  plus parent decision. Never mutates PR branches. Never waits for CI. Works
  with GitHub (`gh`) and GitLab (`glab`). Family: gardener-sow → gardener-tend
  → gardener-harvest.
---

# Gardener Harvest — One-PR Decision

## Scheduling

### Goal

Process **exactly one** oldest eligible non-draft gardener PR: capture evidence,
spawn one assessor when needed, decide `MERGE` / `NEEDS_FIX` / `CLOSE` / `SKIP`,
perform at most one forge mutation, then **exit**. The outer loop may invoke
harvest again. Never walk the whole open-PR snapshot in one parent session.

### Prime invariants

1. **One PR per invocation, then EXIT.** Do not start a second PR.
2. **Never mutate a PR branch.** No rebase, push, force-push, edit, CI repair,
   or manual branch deletion. Allowed forge writes: squash-merge (pinned
   expected-head commands), marked comments, and close.
3. **Never wait for CI.** Pending or failed CI → `SKIP` immediately.
4. **No proxy gates.** No risk levels, confidence percentages, ASK batches,
   file/line size caps, or `repo-rules.yaml`.
5. **Subagent when assessing.** Conflict / pending CI / failed CI skips need
   **no assessor**. Every other decision path that reaches DECIDE must pass the
   shared dispatch gate on a spawned assessor writing `assessment.md`.

### Intent signature

- User invokes `/gardener-harvest` or asks to harvest / merge-queue gardener PRs
- Outer loop schedules harvest alone (see gardener-running)

### When to use

- Ready non-draft gardener PRs need merge / fix-comment / close / temporary skip
- After tend has promoted or repaired a PR

### When NOT to use

- Draft promotion, rebase, CI repair, comment implementation → `gardener-tend`
- Creating a new improvement → `gardener-sow`
- Deep human architectural review of arbitrary PRs → `review` / `deep-review`

### Expected inputs

- `MAIN_REPO` cwd with forge CLI authenticated (`gh` / `glab`)
- Agent pack containing `skills/gardener-sow` (`CONTROL_ROOT`)
- Shared gardener contract + state + providers loaded

### Expected outputs

- At most one gardener PR acted on (merged, commented, closed, or skipped)
- Registries and intent reconciled under `$CONTROL_ROOT/state/gardener/`
- `RUN_DIR` deleted after the invocation finishes
- No PR branch content mutated

### Dependencies

- How to run: `.agents/skills/_shared/runtime/gardener-running.md`
- Contract: `.agents/skills/_shared/runtime/gardener-contract.md`
- State: `.agents/skills/_shared/runtime/gardener-state.md`
- Providers: `.agents/skills/_shared/runtime/providers.md` (Gardener-only ops;
  stub at `.agents/skills/gardener-harvest/resources/providers.md`)
- Dispatch gate: `.agents/skills/_shared/runtime/subagent-dispatch-gate.md`
- Protocol: `.agents/skills/gardener-harvest/resources/execution-protocol.md`
- Assessor prompt: `.agents/skills/gardener-harvest/resources/assessor-prompt.md`
- Grug: `.agents/rules/grug-principles.md`

### Control-flow features

- Gardener identification only (`chore(gardener):` + `gardener/run-` + default
  target + same-repo head; never `gardener/iter-`)
- Oldest non-draft eligible PR only
- Early `SKIP` on conflict / pending CI / failed CI (no assessor)
- `RUN_DIR` capture + one fresh assessor + parent decision
- One SHA recapture max; second mismatch → `SKIP`
- Write-ahead intents: `harvest_merge` / `harvest_needs_fix` / `harvest_close`
- Comment marker `<!-- gardener -->` on every harvest comment

## Structural Flow

### Entry

1. Resolve `MAIN_REPO` (cwd) and `CONTROL_ROOT` (walk parents of this skill file
   until a directory contains `skills/gardener-sow`).
2. Detect `PROVIDER` and verify auth with a functional repo view per
   `.agents/skills/_shared/runtime/providers.md`. Abort on failure.
3. Resolve the remote default branch once. Never hardcode `main`.
4. If `intent.json` exists and `skill` is another gardener skill, **stop** and
   report that skill.
5. **Worktree cleanup** (every gardener skill): `git worktree list`; remove
   `$CONTROL_ROOT/worktrees/gardener-*` paths that are **not**
   `intent.worktree_path` (no intent / empty path → remove all such paths).
6. **Harvest temp cleanup**: delete `$CONTROL_ROOT/results/tmp/pr-harvest/*`
   directories that are **not** `intent.run_dir`. If there is no intent or
   `run_dir` is empty, delete all of them.
7. Validate `closed.json` (stop if present but invalid). Fully paginate open
   PR metadata. Rebuild/reconcile `open.json` for this repository per
   gardener-state. Do not fetch every body.
8. If `intent.json` exists and `skill` is `harvest`, reconcile from actual
   forge/git (protocol), then **exit**. That reconcile (including SHA-changed
   `SKIP`) **is** this invocation. Do not SELECT another PR.

### Scenes

1. **SELECT**: Filter gardener PRs (identification below), `isDraft == false`,
   target == resolved default branch, same-repo head. Sort `createdAt`
   ascending. Take the **oldest** only. If none → exit (nothing to harvest).
2. **EARLY_SKIP**: Decide pending/failed/green from **actual check/pipeline
   status** (`gh pr checks` / pipeline jobs), not from GitHub
   `mergeStateStatus: UNSTABLE` (that shared table row is not gardener
   policy). If conflict/unmergeable, or CI pending/running, or CI failed →
   update open-cache observation for that PR, `SKIP`, **exit**. **Do not
   spawn an assessor.**
3. **CAPTURE**: Allocate `run-id`. Set
   `RUN_DIR=$CONTROL_ROOT/results/tmp/pr-harvest/<run-id>/`. Fetch description,
   patch, exact base/head SHAs, CI summary, and review discussions into
   `RUN_DIR`.
4. **ASSESS**: Spawn one assessor (via `task` / `invoke_subagent`) using
   `.agents/skills/gardener-harvest/resources/assessor-prompt.md`. Assessor
   writes `$RUN_DIR/assessment.md` only. Record harness `task_id` in
   `$RUN_DIR/subagent-ledger.json`.
5. **DISPATCH_GATE**: Before consuming the assessment, require non-empty
   `task_id`, `status == complete`, non-empty `assessment.md`, and required
   section markers (see protocol). On gate failure → re-dispatch once or
   `SKIP` with provider/assessment failure (retain `RUN_DIR`, stop). Never
   decide from inline-only analysis.
6. **PARENT_DECIDE**: Parent independently compares assessment with the actual
   captured patch. Choose `MERGE`, `NEEDS_FIX`, `CLOSE`, or `SKIP`.
7. **RECHECK**: Immediately before any forge mutation, re-fetch base SHA, head
   SHA, mergeability, and CI. If base or head SHA changed → discard assessment,
   recapture + reassess + re-decide **once**. If a SHA still differs after that
   one recapture → `SKIP`, exit.
8. **MUTATE**: For MERGE / NEEDS_FIX / CLOSE, write `intent.json` first, then
   perform the forge effect, verify, update registries, delete intent only when
   the full expected result is verified (protocol).
9. **CLEANUP_EXIT**: Delete `RUN_DIR` after the invocation finishes (except when
   retaining inputs on assessment/provider failure). Exit. Do not select another
   PR.

### Identification (all required)

A gardener PR has **all** of:

- title prefix `chore(gardener):`
- branch prefix `gardener/run-`
- target equal to the resolved default branch
- source branch on the target repository (not a fork)

Do **not** recognize `gardener/iter-` or other prior names.

### Decisions

**MERGE** — all must hold:

- Child verdict is `MERGE` and parent independently agrees
- Valuable coherent micro-improvement; intended correctness preserved/improved
- No unresolved actionable review request remains
- Current CI green; mergeability clean; head SHA unchanged from assessment
- Squash-merge via pinned expected-head commands only:
  - GitHub: `gh pr merge N --squash --delete-branch --match-head-commit <sha>`
  - GitLab: merge API with `squash=true`, `should_remove_source_branch=true`,
    `sha=<head>`
- If that command cannot be issued → do not merge (`SKIP` /
  `merge_strategy_mismatch` as applicable)
- Verify merged state; report branch-deletion failure without manual deletion
- Remove open-cache entry

**NEEDS_FIX**

- Post one concrete actionable forge comment including `<!-- gardener -->`
- Search for the marker and equivalent text first; do not duplicate
- Leave the PR open
- Unresolved actionable review (unless it explicitly rejects/closes) routes here

**CLOSE**

- Child + parent agree nonsense / actively undesirable / not worthwhile, **or**
  an explicit **non-marked** human reject/close comment is authoritative
- Post standardized closure rationale with `<!-- gardener -->`, then close
- After verified close:
  - human non-marked reject/close → `closed.json` category `rejected`,
    `suppress_equivalent: true`
  - agent-only CLOSE → category `unknown`, `suppress_equivalent: false`
- Remove open-cache entry

**SKIP** (temporary; reconsider next run)

- Conflict / unmergeable (early or at recheck)
- Pending or failed CI
- SHA still changed after one recapture
- Provider / assessment / dispatch failure (retain temp inputs; no mutation)
- Expected-head merge cannot be issued

### Transitions

- SELECT finds none → exit clean
- EARLY_SKIP → update open observation → exit
- After MUTATE, SKIP, or finished intent reconcile → cleanup → exit (never
  loop to another PR)

### Failure and recovery

| Failure | Recovery |
|---------|----------|
| Auth / provider detect fail | Stop before mutation |
| Intent owned by another gardener skill | Stop; report that skill |
| Invalid `closed.json` | Stop; never overwrite |
| Invalid / missing `open.json` | Rebuild from gardener metadata |
| Early conflict / CI pending / CI failed | `SKIP`; no assessor |
| Dispatch gate fail | Re-dispatch once or `SKIP`; no inline substitute |
| SHA change | One full recapture+reassess; second mismatch → `SKIP` |
| Merge command cannot pin expected head | Do not merge |
| Uncertain forge response | Query actual state before retrying intent |

### Exit

- Success: one PR merged, needs-fix commented, closed, or skipped; intent clear
  when complete; `RUN_DIR` cleaned when appropriate
- Nothing to harvest: no eligible non-draft gardener PR
- Blocked: foreign intent, invalid closed state, auth failure

## Logical Operations

### Actions

| Action | Evidence |
|--------|----------|
| Resolve paths / provider / default branch | contract + providers |
| Reconcile intent + cleanup pr-harvest dirs | gardener-state |
| List + identify oldest gardener PR | providers + identification |
| Early SKIP on conflict / CI | list/view + checks metadata |
| Capture into `RUN_DIR` | description, diff, SHAs, CI, discussions |
| Spawn assessor | `assessor-prompt.md` → `assessment.md` |
| Dispatch gate | `subagent-dispatch-gate.md` |
| Parent decide | assessment + patch |
| Write intent then mutate | `harvest_merge` / `harvest_needs_fix` / `harvest_close` |
| Update open/closed registries | atomic writes per gardener-state |

### Canonical workflow path

```
1. Resolve MAIN_REPO + CONTROL_ROOT; detect PROVIDER; resolve default branch
2. Stop if foreign gardener intent
3. Cleanup orphan gardener-* worktrees; delete unreferenced pr-harvest/* dirs
4. Validate closed.json (stop if invalid); paginate; reconcile open.json
5. If harvest intent present → reconcile → EXIT (no SELECT)
6. Filter gardener non-draft same-repo default-target; oldest only
7. If conflict OR CI pending/failed → observe + SKIP + EXIT (no assessor)
8. CAPTURE into RUN_DIR; spawn assessor → assessment.md; pass dispatch gate
9. Parent decides; recheck SHAs (one recapture max)
10. Write intent → MERGE / NEEDS_FIX comment / CLOSE → verify → registries
11. Clear intent; delete RUN_DIR; EXIT
```

### Guardrails

1. One PR then EXIT — never snapshot-walk. Do not apply the shared
   multi-merge `sleep 2` default; harvest merges at most once and does not sleep.
2. Never mutate PR branches; never wait for CI. Never delete comments.
3. No risk / confidence / ASK / size / repo-rules gates.
4. No assessor on early conflict/CI skips.
5. Assessor never mutates repo or forge; parent owns decisions and mutations.
6. Every harvest comment includes exact line `<!-- gardener -->`.
7. Agent CLOSE → `unknown` (no suppress). Human non-marked reject → `rejected`
   (suppress).
8. Expected-head merge only; no squash fallback.
9. Capability phrasing for tools (see tool-compatibility); spawn via `task` /
   `invoke_subagent`.
10. Prefer deletion and existing shared machinery (Grug 19, 25).

## References

- `.agents/skills/_shared/runtime/gardener-contract.md`
- `.agents/skills/_shared/runtime/gardener-state.md`
- `.agents/skills/_shared/runtime/gardener-running.md`
- `.agents/skills/_shared/runtime/providers.md`
- `.agents/skills/_shared/runtime/subagent-dispatch-gate.md`
- `.agents/skills/gardener-harvest/resources/execution-protocol.md`
- `.agents/skills/gardener-harvest/resources/assessor-prompt.md`
- `.agents/skills/gardener-harvest/resources/providers.md`
- Sibling skills: `gardener-sow`, `gardener-tend`
