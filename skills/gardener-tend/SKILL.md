---
name: gardener-tend
description: >
  One maintenance action on the oldest gardener PR that needs attention: honor
  human reject/close, rebase, repair PR-attributable CI, address comments, or
  promote a healthy draft. Then exit. Family: gardener-sow → gardener-tend →
  gardener-harvest. Provider-agnostic (`gh` / `glab`).
---

# Gardener Tend — One Maintenance Action

## Scheduling

### Goal

Perform **at most one** maintenance action on the oldest open gardener PR that
needs work, then exit. If there is a backlog, exit **without** sleeping so the
outer loop can start the next shot immediately. Sleep **only** on `all_clear`
(nothing to tend) so idle polling does not burn continuous agent turns.

Load before acting:

- `.agents/skills/_shared/runtime/gardener-contract.md`
- `.agents/skills/_shared/runtime/gardener-state.md`
- `.agents/skills/_shared/runtime/gardener-running.md`
- `.agents/skills/_shared/runtime/providers.md`
- `.agents/skills/_shared/runtime/subagent-dispatch-gate.md`
- `.agents/rules/grug-principles.md`

### Intent signature

- User invokes `/gardener-tend` or asks to tend gardener PRs
- Automated maintenance: rebase, CI repair, comment response, draft promotion
- Explicit human reject/close handling on gardener PRs

### When to use

- Periodic PR health maintenance (outer loop or on-demand)
- After sow opens drafts that need rebase, CI, or comment follow-up
- Before harvest, when drafts need promotion

### When NOT to use

- Creating new micro-improvements → `gardener-sow`
- Assessing/merging ready non-draft gardener PRs → `gardener-harvest`
- Non-gardener PRs, fork PRs, or coordinated architecture work → out of scope
- Deep human-driven review → `review` / `deep-review`

### Expected inputs

- `MAIN_REPO` as cwd (target git repository)
- Forge CLI authenticated for `origin` (`gh` or `glab`)
- Agent pack containing `skills/gardener-sow` (`CONTROL_ROOT`)
- Dirty main checkout is allowed — all edits run in a worktree

### Expected outputs

- Exactly one of: close, rebase push, CI-fix push, comment reply/resolution
  (with optional content push), draft promotion, or a documented no-op exit
- Updated `$CONTROL_ROOT/state/gardener/{open,closed,intent}.json` as required
- No orphaned `gardener-*` worktrees after a completed action
- Main checkout tracked project source unchanged

### Dependencies

- Shared contract/state/providers (paths above)
- Resources under `.agents/skills/gardener-tend/resources/`
- Forge CLI only via `.agents/skills/_shared/runtime/providers.md`
- Local checks only via the Local CI config discovery procedure in providers.md
  (never invent runners; never hardcode a package manager or linter)

### Control-flow features

- One action per invocation; exit after the first action taken
- Priority-ordered selection (see Scenes)
- Intent write-ahead before forge/git mutations; reconcile on entry
- Content changes: worktree + local discovery checks + fresh assessment + at
  most one focused correction
- Comments preserved (reply + resolve); never deleted
- Forks ignored; only same-repo `gardener/run-` PRs targeting the default branch

## Structural Flow

### Entry

1. Resolve `MAIN_REPO` (cwd) and `CONTROL_ROOT` (walk parents of this skill file
   until a directory contains `skills/gardener-sow`).
2. Detect `PROVIDER` from `origin`; verify auth with a functional repo view
   (`gh repo view` / `glab repo view`). Resolve the default branch once. On
   GitLab, resolve `:pid` once.
3. If `intent.json` exists and `skill` is not `tend` → stop; report owning skill.
4. Worktree cleanup: `git worktree list`; remove paths under
   `$CONTROL_ROOT/worktrees/` matching `gardener-*` that are **not**
   `intent.worktree_path` (no intent / empty path → remove all such paths).
5. Validate/rebuild gardener state per `gardener-state.md` (paginated open
   gardener metadata; never overwrite invalid `closed.json`).
6. If `skill` is `tend` and `intent.json` exists → reconcile from actual
   git/forge (execution-protocol.md), then **exit**. That reconcile **is**
   this invocation. Do not walk PRs.
7. List open PRs (full pagination). Keep only PRs that match **all** of:
   - title prefix `chore(gardener):`
   - branch prefix `gardener/run-`
   - `baseRefName` equals the resolved default branch
   - same-repository head (not a fork)
   Sort by `createdAt` ascending. Sync `open.json` from this metadata set.

### Scenes

Walk oldest-first. For each PR, evaluate checks in this **strict priority**.
On the first actionable item, perform that single action and exit.

1. **HUMAN_REJECT** — non-marked comment explicitly asks to reject or close
   → `tend_close` (authoritative).
2. **REBASE** — conflicts or behind base → `tend_rebase`.
3. **CI_REPAIR** — decide from **actual check/pipeline status**, not from
   GitHub `mergeStateStatus: UNSTABLE` (that is not “running — skip”).
   - Failed checks **attributable to this PR** → `tend_ci`.
   - Pending/running CI → skip this PR (continue the oldest-first walk).
   - Unrelated / base / infrastructure failures → post a marked comment
     explaining no PR change; that comment **is** the one action; **exit**.
     Do not fold those failures into the PR.
4. **COMMENTS** — other actionable reviewer comments → `tend_comment`.
   Vague comments → reply asking clarification (still one action), exit.
5. **PROMOTE** — draft, mergeable, CI complete+green, no actionable unresolved
   comments, fresh assessment of exact head passes → `tend_promote`.
6. **ALL_CLEAR** — no PR needed action → report `all_clear`, **`sleep 300`**,
   then exit. Do **not** sleep after a real tend action (close / rebase / CI /
   comment / promote) — exit immediately so a queue drains fast.

### Transitions

- Own-skill intent reconcile (when present) wins → finish it → exit.
- First actionable PR+check wins → act → exit.
- Content-changing actions (rebase, CI repair, comment implementation): after
  pushable patch is ready → discovery local checks → fresh assessment → on
  FAIL allow **one** focused correction + full re-verify/reassess → on second
  FAIL stop without push.
- After any successful content push, reassess whether further tend work is
  needed on a **later** invocation (this invocation already spent its action).
- Never mix close + push + promote in one invocation.

### Failure and recovery

| Failure | Recovery |
|---------|----------|
| Auth / provider detect fails | Stop before mutation |
| Intent owned by another gardener skill | Stop; report that skill |
| Invalid `closed.json` | Stop; do not overwrite |
| Open state absent/invalid/wrong repo | Rebuild from forge metadata |
| Worktree setup fails | Stop without source mutation |
| Remote head changed vs captured SHA | Abort push; do not overwrite |
| Local checks fail | Stop without push (failures are allowed) |
| Assessment FAIL after one correction | Stop without push |
| Unrelated CI failure | Marked gardener comment explaining skip; no code change |
| Rate limit (429) | Save state; exit |
| Cleanup failure | Report exact retained path; never claim clean completion |

### Exit

- Success: one tend operation completed and intent cleared after verification.
- No-op: no gardener PR needed action.
- Failure: blocker reported; no silent partial registry writes.

## Logical Operations

### Actions

| Action | SSL primitive | Evidence |
|--------|---------------|----------|
| Resolve paths / provider / default branch | `READ` | origin, skill parents, repo view |
| Reconcile intent | `COMPARE` | intent.json vs forge/git |
| Reconcile open registry | `UPDATE_STATE` | paginated gardener PR metadata |
| Select oldest actionable PR | `SELECT` | priority checks |
| Write tend intent | `WRITE` | intent.json before mutation |
| Worktree add/remove | `CALL_TOOL` | `$CONTROL_ROOT/worktrees/gardener-<run-id>` |
| Non-interactive rebase / commit | `CALL_TOOL` | GIT_EDITOR bypasses |
| Local discovery checks | `VALIDATE` | providers.md Local CI discovery |
| Spawn implement/verify/assess/revise | `CALL_TOOL` | `task` + dispatch gate |
| Push with lease | `CALL_TOOL` | `--force-with-lease` + captured SHA |
| Reply / resolve comments | `CALL_TOOL` | marked `<!-- gardener -->` body |
| Close + registries | `UPDATE_STATE` | closed rejected + remove open |
| Promote draft | `CALL_TOOL` | ready when promotion gate passes |

### Canonical workflow path

```
1. Entry: paths, provider, auth, default branch
2. Stop if another gardener skill owns intent
3. Cleanup orphan gardener-* worktrees; reconcile open.json (stop if closed.json invalid)
4. If tend intent exists → reconcile from git/forge → EXIT (do not walk)
5. List same-repo gardener/run- PRs targeting default; oldest first
6. Priority walk → one of: close | rebase | ci | comment | promote | no-op
7. For mutations: write intent → perform → verify forge/git → update registries → clear intent
8. Exit
```

### Resource scope

| Scope | Target |
|-------|--------|
| `CODEBASE` | PR branch only, via `WORKTREE` |
| `LOCAL_FS` | `$CONTROL_ROOT/state/gardener/`, worktrees, temp ledgers |
| `PROCESS` | forge CLI, discovered local check commands, git |
| `NETWORK` | forge API via CLI |
| `MEMORY` | intent, open/closed registries, subagent ledger |

### Preconditions

- Functional forge auth against the repo
- Identification rules from gardener-contract.md
- Content edits only in `WORKTREE`

### Effects and side effects

- May force-with-lease push the PR branch
- May reply to / resolve comments (never delete)
- May close a PR and write `rejected` closed learning
- May promote draft → ready
- May create/remove gardener worktrees under `CONTROL_ROOT`

### Guardrails

1. **One action only** — then exit.
2. **Gardener identity** — title + `gardener/run-` + default target + same-repo; forks out of scope.
3. **Never delete comments** — reply with outcome; resolve when supported.
4. **Every gardener comment** includes the exact line `<!-- gardener -->`.
5. **Human reject/close** (non-marked explicit ask) is authoritative → close, write
   `closed.json` category `rejected` with `suppress_equivalent: true`, remove open entry.
6. **Agent CLOSE comments are marked** and must not suppress.
7. **PR-attributable CI only** — do not absorb unrelated failures.
8. **Capture head SHA** before content work; push `--force-with-lease`; abort on unexpected remote head.
9. **Non-interactive git only** — `GIT_EDITOR=true` / `GIT_SEQUENCE_EDITOR=true` (or equivalent).
10. **Local checks** via discovery procedure only — no Poetry/Ruff/coverage hardcoding.
11. **Idle sleep only** — `sleep 300` solely on `all_clear`. Never sleep after
    a completed tend action.
12. **Verification can fail** — stop without push; never claim verification cannot fail.
13. **Dispatch gate** — delegated implement/verify/revise/assess outputs must pass
    `.agents/skills/_shared/runtime/subagent-dispatch-gate.md` before consumption.
14. **Do not require** any particular project guidance filename by name.
15. **Intent ops only**: `tend_rebase`, `tend_ci`, `tend_comment`, `tend_close`, `tend_promote`.

## References

- Contract: `.agents/skills/_shared/runtime/gardener-contract.md`
- State: `.agents/skills/_shared/runtime/gardener-state.md`
- Running: `.agents/skills/_shared/runtime/gardener-running.md`
- Providers + Local CI discovery: `.agents/skills/_shared/runtime/providers.md`
- Dispatch gate: `.agents/skills/_shared/runtime/subagent-dispatch-gate.md`
- Execution: `.agents/skills/gardener-tend/resources/execution-protocol.md`
- Worktrees: `.agents/skills/gardener-tend/resources/worktree-isolation.md`
- Assessment prompt: `.agents/skills/gardener-tend/resources/assessment-prompt.md`
- Implement prompt: `.agents/skills/gardener-tend/resources/implement-prompt.md`
- Revise prompt: `.agents/skills/gardener-tend/resources/revise-prompt.md`
- Scenarios: `.agents/skills/gardener-tend/resources/scenarios.md`
