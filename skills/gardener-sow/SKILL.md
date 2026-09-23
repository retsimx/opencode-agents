---
name: gardener-sow
description: >
  Single-shot gardener sow: reconcile state, propose one evidence-backed
  micro-improvement in an isolated worktree, implement, verify, assess, open
  one draft PR, then exit. No user interaction. Thin orchestration — substantive
  work via task subagents. Outer shell loops for continuity. Family:
  gardener-sow → gardener-tend → gardener-harvest.
---

# Gardener Sow — One Draft Micro-Improvement

Load before acting:

- `.agents/skills/_shared/runtime/gardener-contract.md`
- `.agents/skills/_shared/runtime/gardener-state.md`
- `.agents/skills/_shared/runtime/providers.md` (Gardener-only forge ops + Local CI discovery)
- `.agents/skills/_shared/runtime/subagent-dispatch-gate.md`
- `.agents/rules/grug-principles.md`

How to run: `.agents/skills/_shared/runtime/gardener-running.md`.

## Goal

One coherent, evidence-backed draft gardener PR per invocation — or a clean
skip/abandon — then end the turn. Never loop inside this skill.

## Paths (resolve once)

| Name | Value |
|------|--------|
| `MAIN_REPO` | Target git repository (cwd / forge + `git worktree add`) |
| `CONTROL_ROOT` | Walk parents of this skill file until a dir contains `skills/gardener-sow` |
| `STATE_DIR` | `$CONTROL_ROOT/state/gardener` |
| `run-id` | Unique id (e.g. UTC timestamp + short suffix) |
| `WORKTREE` | `$CONTROL_ROOT/worktrees/gardener-<run-id>` |
| `BRANCH` | `gardener/run-<run-id>` |
| `RUN_TMP` | `$CONTROL_ROOT/results/tmp/gardener-sow-<run-id>/` |
| `LEDGER` | `$RUN_TMP/subagent-ledger.json` |

Do not edit tracked project source in the main checkout. Source edits and checks
run in `WORKTREE`. Registry/intent writes under `CONTROL_ROOT` are allowed.

## Orchestrator role

Thin runner only: resolve paths, reconcile intent/state, create/cleanup worktrees,
spawn subagents, enforce the dispatch gate, write `intent.json`, update
`open.json`, delete temps. Do **not** scan, implement, verify, assess, or write
PR bodies inline.

Every substantive subagent (scanner, implement, verify, assessor, revise, ship)
must pass `.agents/skills/_shared/runtime/subagent-dispatch-gate.md` before the
next stage consumes its output: non-empty harness `task_id` in `LEDGER`,
`status == complete`, non-empty `result_file`, required section markers present.

Pass absolute `MAIN_REPO`, `CONTROL_ROOT`, `WORKTREE` (once created), `PROVIDER`,
`DEFAULT_BRANCH`, and `RUN_TMP` to every subagent. Shell commands in subagents:
wrap with `timeout 300`.

## Flow (exactly once)

### 0. Entry

1. Detect `PROVIDER` and verify auth via functional repo view (`providers.md`).
2. Resolve `DEFAULT_BRANCH` (never hardcode `main`).
3. If `$STATE_DIR/intent.json` exists and `skill` is another gardener skill,
   **stop** and report it.
4. **Cleanup orphan worktrees**: `git worktree list`; remove paths under
   `$CONTROL_ROOT/worktrees/` matching `gardener-*` that are **not**
   `intent.worktree_path` (if no intent / empty path, remove all such paths).
5. **Reconcile `open.json`**: fully paginate open gardener PR metadata; rebuild
   if absent/invalid/wrong repo (shared protocol
   `.agents/skills/_shared/runtime/gardener-state.md`). Validate `closed.json`
   (stop if present but invalid).
6. If `intent.json` exists and `skill` is `sow`, reconcile `ship_draft_pr`
   from actual git/forge (see Interrupt recovery), then **exit** (reconcile
   consumes the invocation — `gardener-state.md`). If absent, continue.
7. Allocate `run-id`. Create `RUN_TMP` and empty `LEDGER`.
8. Create worktree:  
   `git -C "$MAIN_REPO" fetch origin "$DEFAULT_BRANCH"`  
   `git -C "$MAIN_REPO" worktree add -b "$BRANCH" "$WORKTREE" "origin/$DEFAULT_BRANCH"`

### 1. Scanner → proposal

Spawn task with `resources/scanner-prompt.md`.

- Result: `$RUN_TMP/proposal.md` with required markers (see prompt).
- Gate, then read. If `NO_PROPOSAL` / empty worthwhile work → cleanup, exit.

### 2. Parent suppression check

Per shared `gardener-state.md`: walk open items and closed `rejected` items
only. Suppress only if a scanner-cited **open or `rejected`** row is the same
concern and the parent agrees, or the parent finds a missed colliding
open/`rejected` row. A cited `unknown` row does **not** suppress. On suppress
→ cleanup, exit (no source ship).

### 3. Implementation preflight + implement

Spawn task with `resources/implement-prompt.md` (preflight then edit).

- Preflight must validate proposal against current `WORKTREE` code. Invalid →
  cleanup, exit.
- Implement exactly one improvement. Result: `$RUN_TMP/implement.md`. Gate,
  then require `STATUS: DONE`. `STATUS: REJECT` → cleanup, exit. Do not
  continue on missing or non-DONE status.

### 4. Local checks

Spawn task with `resources/verify-prompt.md`.

- Discover checks per contract / `providers.md` Local CI discovery. Never invent
  runners or install tooling.
- Result: `$RUN_TMP/verify.md`. Gate, then read `STATUS`:
  - `FAIL` → revision path (or abandon if already revised once).
  - `PASS` or `NO_LOCAL_SUITE` → continue (no local suite is not a failure;
    rely on forge checks + tend).
  - Any other status → abandon, no ship.

### 5. Assessor

Spawn task with `resources/assessor-prompt.md`.

- Result: `$RUN_TMP/assessment.md` with `PASS` or `FAIL`. Gate.
- Parent independently skims the **uncommitted** `$WORKTREE` patch
  (`cd "$WORKTREE"`; `git status` / `git diff` / `git diff --cached`; open
  `CHANGED` paths from `implement.md`). Must agree before ship. Do **not**
  use `origin/<default>...HEAD` here — assess is before commit; an empty
  committed range is not “no implementation.”

### 6. One revision (at most)

On verify fail or assessor `FAIL`: spawn `resources/revise-prompt.md` once, then
re-run verify + assessor. Second failure → abandon (revert worktree dirty state
if needed), cleanup, exit.

### 7. Ship

1. Spawn `resources/ship-prompt.md` **MODE=COMMIT**: one local commit
   `chore(gardener): …`, write `$RUN_TMP/pr-body.md` from
   `resources/pr-body-template.md` + verified patch. Return commit SHA. Gate,
   then require `STATUS: COMMITTED` and a non-empty `HEAD_SHA`. On `FAIL` or
   missing SHA → do **not** write intent; do not publish; cleanup, exit.
2. Atomically write `$STATE_DIR/intent.json` with
   `skill=sow`, `operation=ship_draft_pr`, `worktree_path`, `run_id`, branch,
   `head_sha`, `base_sha`, title, PR body path, expected forge result.
3. Spawn ship **MODE=PUBLISH**: push branch; create gardener draft targeting
   `DEFAULT_BRANCH` (Gardener-only create row in `providers.md`, never the
   shared `--base main` create row). Gate, then require `STATUS: SHIPPED` and
   a PR number.
4. Verify PR exists; update `open.json` (number, url, title, branch, head_sha,
   topic, rationale, areas, created_at).
5. Delete `intent.json` only after expected result is verified.
6. Cleanup: remove `WORKTREE` + branch local ref as appropriate; **retain** `RUN_TMP` (session artifacts are the audit trail and are not auto-deleted).

### 8. Exit

End the turn. Do not start another sow.

## Interrupt recovery (`ship_draft_pr`)

Query remote branch / PR carrying the recorded `run-id`:

| Remote state | Action |
|--------------|--------|
| Branch pushed, no PR | Create the one intended draft; then open.json; clear intent; **exit** |
| PR exists | Record in open.json; clear intent; **exit** |
| Neither | Report failed ship; clean abandoned worktree/branch; clear intent only after confirming no external effect; **exit** |
| Differs from intent | Stop and report — do not guess; **exit** |

Do not fall through to allocate a new `run-id` or open a second draft.

## PR body

Sections: **Improvement**, **Safety**, **Verification**, **Grug**, optional
**Prior proposal**. See `resources/pr-body-template.md`. Examples:
`resources/acme-examples.md`.

## Guardrails

1. Single shot; no in-skill loop; no `question` tool.
2. No inline substitution for required subagents (dispatch gate).
3. No Poetry / Ruff / `test_coverage.sh` / `main` / `AGENTS.md` / `TESTING.md`
   hardcoding — discover guidance and checks.
4. No `gardener/iter-`. Do not use a legacy Markdown state *table*. The shared
   protocol `.agents/skills/_shared/runtime/gardener-state.md` is required.
5. Draft PRs only; title `chore(gardener):`; branch `gardener/run-…`.
6. Exclusions: `resources/exclusions.md` (secrets / generated / vendored only).
7. Do not touch excluded paths; do not invent local CI.

## Resources

| File | Role |
|------|------|
| `resources/scanner-prompt.md` | Evidence-backed proposal |
| `resources/implement-prompt.md` | Preflight + implement |
| `resources/verify-prompt.md` | Local check discovery |
| `resources/assessor-prompt.md` | Independent patch assessment |
| `resources/revise-prompt.md` | One focused revision |
| `resources/ship-prompt.md` | Commit / publish |
| `resources/pr-body-template.md` | Public PR body |
| `resources/acme-examples.md` | Good/bad proposal examples |
| `resources/exclusions.md` | Generic path exclusions |
