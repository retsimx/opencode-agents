---
name: gardener-harvest
description: >
  Sequential assessment and merge of open PRs via subagent risk classification,
  behavior-change detection, CI detail reporting, change-size measurement,
  confidence scoring, and repository-specific risk globs. Pure read-only against
  PRs — never mutates a PR. Merges (LOW risk + green CI + high confidence),
  asks the user (MEDIUM or uncertain), or skips (HIGH risk, failing or pending CI,
  draft, large diff). Works with GitHub (`gh`) and GitLab (`glab`). Family:
  gardener-sow → gardener-tend → gardener-harvest.
---

# Gardener Harvest — Sequential Assessment and Merge

## Scheduling

### Goal
Walk open PRs in chronological order, assess each one via a subagent task that
performs risk-based classification (LOW/MEDIUM/HIGH), behavior-change detection,
CI detail reporting, change-size measurement, confidence scoring, and
repository-specific rule checking. Merge only when every gate passes; ask the
user otherwise; skip drafts, pending/failed CI, large diffs, and known-problem
PRs. Continue until every open PR has been processed. Persist per-run state so an
interrupted run can resume.

### Prime invariant (non-negotiable)

**NEVER mutate a PR's branch or contents. Do not rebase. Do not investigate rebase conflicts. Do not
fix failing CI. Do not push anything to the PR branch. Do not edit files in the
repo. The only writes the skill performs are (a) `.agents/results/pr-merge-queue.json`
and (b) the squash-merge itself, which the user explicitly authorized by running
the skill.**

A merge retry is NOT a mutation. Re-issuing the same
`gh pr merge <n> --squash --delete-branch` (or glab equivalent) after a transient
provider failure is fully within the user's authorization and is the skill's
intended behaviour. What is forbidden is editing the branch's contents to make a
merge succeed.

When something is permanently wrong with a PR (real conflict, branch protection
rejects squash, PR changed since assessment, CI failing) skip it and move on. No
exception, including for the user's own PRs, gardener PRs, or "just one tiny fix"
cases. PR maintenance is the `gardener-tend` skill's job, not this skill's.

### Intent signature
- User asks to merge open PRs in order
- User wants a sequential review-and-merge pass over all open PRs
- User mentions "merge queue", "merge all PRs", "harvest PRs", or "process open PRs"
- User invokes `/gardener-harvest`

### When to use
- Several open PRs are queued and the user wants them merged one by one
- PRs are expected to be small focused changes (imports, comments, tests, refactors)
- CI is the primary correctness signal and the diff is the primary quality signal

### When NOT to use
- A single PR with a known issue -> use `debug` skill
- PRs need deep architectural review -> use `review` or `deep-review` skill
- PRs are draft or WIP and need promotion -> use `gardener-tend`
- CI is failing and the user wants it fixed -> use `gardener-tend`
- Creating new micro-fix PRs -> use `gardener-sow`
- Merging requires human sign-off per PR -> out of scope

### Expected inputs
- Local repo with `gh` (GitHub) or `glab` (GitLab) CLI authenticated and on `PATH`
- Open non-draft PRs exist on the default branch
- CI is configured and runs on PRs

### Expected outputs
- Each PR either merged (source branch deleted by provider) or parked with a reason recorded in `.agents/results/pr-merge-queue.json`
- A final summary listing every PR and its outcome
- No PR branch mutated, no PR rebased, no CI touched

### Dependencies
- `gh` CLI (GitHub) or `glab` CLI (GitLab), authenticated and on `PATH`
- `task` tool to spawn subagent assessment tasks
- `question` tool to batch uncertain PRs to the user
- How to run (outer loops): `../_shared/runtime/gardener-running.md`
- `resources/providers.md` — stub → shared provider command map (`../_shared/runtime/providers.md`)
- `../_shared/runtime/providers.md` — GitHub (`gh`) / GitLab (`glab`) detection and commands
- `resources/repo-rules.yaml` — high-risk file globs for this project
- `resources/execution-protocol.md` — state file, batching, rate limits, large diffs
- `resources/subagent-prompt.md` — exact prompt launched subagents use

### Control-flow features
- Sequential merge gates in oldest-first creation-date order
- Parallel assessment batching (default batch size 5; see `resources/execution-protocol.md`)
- State persistence in `.agents/results/pr-merge-queue.json` for resume
- Batched ASK presentation in a single `question` call with `multiple: true`
- Provider abstraction (GitHub `gh` / GitLab `glab`)
- Repository-specific risk globs loaded from `resources/repo-rules.yaml`
- Squash merge + delete source branch (fixed, not configurable)
- Never mutates a PR — skips on any PR-side issue

## Structural Flow

### Entry
1. Detect provider from `git remote get-url origin` per `resources/providers.md`. Abort if neither `gh` nor `glab` is authenticated for the resolved provider.
2. Load (or create) `.agents/results/pr-merge-queue.json` per `resources/execution-protocol.md`.
3. Load `resources/repo-rules.yaml` and prepare the high-risk glob list.
4. Confirm at least one open PR exists after draft and resume filtering.

### Scenes
1. **LIST**: Fetch ALL open PRs via the provider's list command with an explicit `--limit` (GitHub: `--limit 1000`, or paginate via `gh api search/issues` above 1000 — see `resources/providers.md`). Capture `number`, `title`, `headRefName`, `createdAt`, `isDraft`. The `gh pr list` default limit of 30 is NEVER acceptable. Re-sort client-side by `createdAt` ascending. Filter drafts (`isDraft == false`) and any PR number already in the state file's `processed` list. Log the remaining count to the user before proceeding.
2. **PREFILTER**: For each PR in oldest-first order, before spawning any subagent:
   - Skip if the PR number is already in `processed` (state file resume).
   - Fetch `gh pr diff <number>` (or `glab mr diff <number>`). If the diff output exceeds ~50,000 chars, record `skipped: large_diff` and continue. Never attempt to summarize.
3. **BATCH_ASSESS**: Take the next up-to-5 unprocessed PRs. Spawn one subagent per PR using `resources/subagent-prompt.md`, substituting `<number>`, `<repo_path>`, `<provider>`, and `<high_risk_globs>`. Subagents report raw findings only — no merge recommendation, no decision logic.
4. **DECIDE**: For each report received (in oldest-first order), apply the decision logic:
   - **MERGE**: risk = LOW AND behavior = NO AND CI = all passed AND filesChanged ≤ 10 AND total lines changed ≤ 300 AND no repository-specific flags AND confidence ≥ 90%
   - **ASK**: risk = MEDIUM OR behavior = UNCLEAR OR confidence < 90% OR any repository-specific flag present OR scope coherence = NO
   - **SKIP**: risk = HIGH OR CI has any failure OR any check pending OR files changed > 20 OR obvious bug visible in diff OR security concern OR merge strategy mismatch
   - CI pending alone is a SKIP (reason `ci_pending`) — never wait, never poll, never re-check. Move on immediately.
5. **MERGE**: For each MERGE decision, issued SERIALLY in oldest-first order (never parallel — see `resources/execution-protocol.md`):
   a. Run `gh pr merge <number> --squash --delete-branch` (GitHub) or `glab mr merge <number> --squash --remove-source-branch` (GitLab).
   b. If the merge command fails or the subsequent state check is not `MERGED`, apply the Merge Retry Policy in `resources/execution-protocol.md`:
      - Transient failures (405, 409 with policy lag, 429, 5xx, "try again" CLI messages, eventual-consistency false negatives): retry up to 3 times with backoff (5s, 15s), re-checking state and head SHA before each retry.
      - Permanent failures (conflict, branch protection rejects squash, PR changed since assessment, PR disappeared): record `skipped: <reason>` and continue. No retry.
   c. `sleep 2` between successful merges to let the provider's branch-protection state machine settle.
   d. Never rebase, push, force-push, edit files, run CI locally, change merge strategy, or delete branches by hand — even during retry. Retrying the same `--squash --delete-branch` command is the only permitted retry action.
6. **VERIFY**: After a successful merge (or transient retry that succeeds), verify via `resources/execution-protocol.md`:
   - PR state is `MERGED`.
   - Branch ref is gone (404). If still present, record `outcome: merged_branch_not_deleted` and continue. **Do not** delete the branch manually.
7. **BATCH_ASK**: After all MERGE and SKIP decisions in the pass are resolved, collect all ASK-classified PRs. Present them in a single `question` tool call with `multiple: true`, chunked into ≤10 per call if more than 10 accumulate. Each option label is `#<n> <title>` and the description is the subagent report's `Top concerns` + `Confidence` + `Risk` summary. User picks which to proceed with.
8. **MERGE_ASKED**: For each PR the user selected, run step 5–6 (merge + verify), in oldest-first order.
9. **FINALIZE**: Append every ASK-without-proceed to `processed` as `skipped: user_skipped_ask`. Output a summary table with columns `PR | Risk | Behavior change | CI | Files/Lines | Confidence | Outcome | Reason`. Save state file.

### Transitions
- If a PR is MERGE -> run step 5-6, record outcome in state, next PR.
- If a PR is ASK -> defer to BATCH_ASK; after user selects, merge selected and record the rest as skipped.
- If a PR is SKIP -> record outcome + reason in state, next PR.
- If subagent task times out or fails -> record `skipped: subagent_failed`, continue.
- If the provider returns 429 / rate-limited -> finish pending batch, save state, exit with reason `rate_limited`.
- **Never** escalate a SKIP into an investigation. There is no recovery path that mutates a PR.

### Failure and recovery
| Failure | Recovery |
|---------|----------|
| Provider not authenticated | Exit with error before listing |
| No open PRs | Exit with "no open PRs" message |
| Subagent task fails or times out | Record `skipped: subagent_failed`, continue |
| Merge transient failure (405, 409 policy lag, 429 merge-route, 5xx) | Retry merge up to 3 times with backoff (5s, 15s), per `resources/execution-protocol.md`. Re-check state + head SHA before each retry. If state already `MERGED`, treat as success. If head SHA changed, record `skipped: pr_changed_after_assessment`. |
| Merge permanent failure (real conflict, branch protection rejects squash, PR disappeared) | Record `skipped: <specific reason>`, continue. Never rebase. Never edit. Never change merge strategy. |
| Branch not deleted after merge | Record `merged_branch_not_deleted`, continue. Never delete manually. |
| CI still pending | Record `skipped: ci_pending`, continue. Never wait. Never re-check. |
| CI failed | Record `skipped: ci_failed`, continue. Never investigate. |
| Large diff (>50k chars) | Record `skipped: large_diff`, continue. Never summarize. |
| Provider rate-limited (429 outside merge route) | Finish current batch, save state, exit. Never retry within the same run. |
| Rate-limited on the merge route specifically | Covered by the merge retry policy — backoff and retry merge. Distinguish from general rate-limit by inspecting which endpoint returned 429. |

### Exit
- Success: all open PRs processed, each either merged or parked with a recorded reason in `.agents/results/pr-merge-queue.json`. Summary table emitted.
- Partial success: some PRs merged, others parked; user was asked about ASKs in a batch; state file is intact for resume.
- Failure: provider not authenticated, no open PRs, or rate-limited before any work could complete.

## Logical Operations

### Actions
| Action | SSL primitive | Evidence |
|--------|---------------|----------|
| Detect provider | `READ` | `git remote get-url origin` |
| Verify CLI auth | `CALL_TOOL` | `gh auth status` / `glab auth status` |
| List open PRs | `CALL_TOOL` | per `resources/providers.md` |
| Filter drafts and large diffs | `SELECT` | client-side predicate on PR metadata |
| Load state file | `READ` | `.agents/results/pr-merge-queue.json` |
| Load repo rules | `READ` | `resources/repo-rules.yaml` |
| Spawn assessment batch | `CALL_TOOL` | `task` subagents (up to 5 parallel) |
| Collect findings | `INFER` | subagent reports |
| Apply decision logic | `SELECT` | MERGE / ASK / SKIP classification |
| Merge PR (serial) | `CALL_TOOL` | `gh pr merge --squash --delete-branch` / `glab mr merge --squash --remove-source-branch`, one at a time, `sleep 2` between merges |
| Retry transient merge failure | `CALL_TOOL` | up to 3 retries with backoff (5s, 15s), re-check state + head SHA before each, per `resources/execution-protocol.md` |
| Verify merge + branch deletion | `CALL_TOOL` | `gh pr view --json state`; `gh api .../branches/{branch}` |
| Save state | `WRITE` | `.agents/results/pr-merge-queue.json` |
| Ask user | `REQUEST` | `question` tool, batched, `multiple: true` |
| Report summary | `NOTIFY` | final outcome table |

### Tools and instruments
- `gh` CLI (GitHub) — see `resources/providers.md`
- `glab` CLI (GitLab) — see `resources/providers.md`
- `task` tool for parallel subagent assessment
- `question` tool for batched user confirmation
- `rg` / `git log --grep` for repository history evidence (inside subagent)

### Canonical workflow path

```
1. Detect provider from `git remote get-url origin`.
2. Verify CLI auth via `gh auth status` / `glab auth status`.
3. Load `.agents/results/pr-merge-queue.json` (create if missing).
4. Load `resources/repo-rules.yaml` and flatten the high_risk_globs list.
5. List ALL open PRs via provider's list command. MUST pass an explicit limit:
   - GitHub: `gh pr list --state open --json number,title,headRefName,createdAt,isDraft --limit 1000`
     (paginate via `gh api search/issues` above 1000 — see `resources/providers.md`)
   - GitLab: `glab mr list --state opened --output json --per-page 100 --page N` until empty
   Re-sort client-side by `createdAt` ascending. Filter `isDraft == false`.
6. PREFILTER oldest-first:
   - skip already-processed (state file)
   - skip drafts
   - fetch diff; if > 50k chars, record `skipped: large_diff`
7. Form a batch of unfiltered PRs (oldest first). See `resources/execution-protocol.md`
   for scale rules: 1–100 PRs => batch 5; 101–500 => batch 3; >500 => batch 2.
8. Spawn one `task` subagent per PR using `resources/subagent-prompt.md`,
   substituting:
     <number>, <repo_path>, <provider>, <high_risk_globs>
9. Collect subagent reports. Apply decision logic per the DECIDE scene.
10. For each MERGE in oldest-first order, SERIALLY (never in parallel):
      gh pr merge <n> --squash --delete-branch
        # or
      glab mr merge <n> --squash --remove-source-branch
    Then `sleep 2` before the next merge.
    On transient failure (405/409-lag/429-merge-route/5xx): apply the Merge Retry Policy
    in `resources/execution-protocol.md` — retry up to 3 times with backoff (5s, 15s),
    re-checking state + head SHA before each retry.
    On permanent failure (real conflict, branch protection, PR changed, disappeared):
    record `skipped: <reason>`, continue. No rebase. No edit. No strategy change. No retry.
11. VERIFY per `resources/execution-protocol.md` (state MERGED + branch 404).
12. Record outcomes in state file. Continue fetching next batch (step 7).
13. After all batches: collect all ASK PRs. Present them in a single `question`
    call (chunk ≤ 10 each) with `multiple: true`. User selects.
14. For each selected ASK, run step 10-11 and record outcome.
15. Record unselected ASKs as `skipped: user_skipped_ask`.
16. Emit summary table. Save state. Exit.
```

### Resource scope
| Scope | Resource target |
|-------|-----------------|
| `CODEBASE` | PR diffs, CI pipeline outputs, repo-rules.yaml |
| `LOCAL_FS` | `.agents/results/pr-merge-queue.json` (the only local mutation) |
| `PROCESS` | `gh` CLI or `glab` CLI, `task` subagent tasks |
| `MEMORY` | PR metadata, assessment results, decision state |
| `NETWORK` | GitHub REST API via `gh` or GitLab REST API via `glab` |

### Preconditions
- `gh` or `glab` is installed, authenticated, and on `PATH` for the resolved provider.
- At least one open, non-draft PR exists.
- CI is configured for the repo (otherwise CI counts will be 0 passed / 0 failed / 0 pending — the parent must treat "no checks observed" as ASK with `reason: ci_unknown`, not as a free pass).
- The user has write permission sufficient to merge + delete source branch on the resolved provider.

### Effects and side effects
- PRs that pass every gate are squash-merged with the source branch deleted. This is the only PR-side mutation the skill performs, and it is the skill's whole purpose.
- State is persisted to `.agents/results/pr-merge-queue.json`. This is the only filesystem mutation.
- Subagent tasks are spawned (up to 5 in parallel per batch).
- The user may be interrupted once per pass with a batched `question` covering all ASK PRs.
- The skill never: pushes to a PR branch, rebases a branch, force-pushes, edits files in the repo, deletes a branch manually when `--delete-branch` failed, runs CI locally, commits anything.

### Guardrails
1. **Never mutate a PR's branch or contents. No rebase. No CI fix. No conflict investigation. No file edits. No retry-on-fix.** A merge retry (re-issuing the same approved `gh pr merge --squash --delete-branch` command after a transient provider failure) is explicitly NOT a mutation and is allowed by the Merge Retry Policy in `resources/execution-protocol.md`. This guardrail overrides any other rule below if they ever conflict.
2. **Never wait for pending CI.** CI pending → SKIP with `reason: ci_pending`. Move on immediately.
3. **Never delete a branch by hand.** Trust `--delete-branch` / `--remove-source-branch`. If the branch is still present after merge, record it and move on.
4. **Never rely on the provider's default PR list limit.** Always pass `--limit 1000` (or paginate) for GitHub, and explicitly paginate `--per-page 100 --page N` for GitLab. The `gh pr list` default of 30 is a silent failure mode.
5. **Never assume drafts are excluded.** Filter `isDraft == false` client-side, on both providers. GitHub's `gh pr list` does NOT exclude drafts by default despite what some docs claim.
6. **Auto-merge requires every gate.** LOW risk + no behavior change + all CI passed + ≤10 files + ≤300 lines + no repo flags + confidence ≥ 90%. One miss → ASK or SKIP.
7. **No separate CI between required and optional.** Any failed check (Y > 0) → SKIP with `reason: ci_failed`.
8. **Squash merge only.** If the provider rejects squash, SKIP with `reason: merge_strategy_mismatch`. Do not fall back to merge commits or rebase.
9. **Delete branch on merge.** Required flag. Non-configurable.
10. **Oldest first.** Always merge in creation-date order, even when subagent reports arrive out of order.
11. **Merges are serial, never parallel.** Issue one `gh pr merge` / `glab mr merge` at a time with a `sleep 2` gap. Parallel merges cause transient 405/409/5xx failures on the provider's merge route.
12. **Retry only transient merge failures, never permanent ones.** Transient (405, 409-policy-lag, 429-merge-route, 5xx, eventual-consistency false negative): retry up to 3 times with backoff (5s, 15s), re-checking state + head SHA before each retry. Permanent (real conflict, branch protection rejects squash, PR changed since assessment, PR disappeared): skip with `skipped: <reason>`. Never rebase, edit, or change strategy to convert a permanent failure into a retry.
13. **Scale the batch size with PR count.** 1–100 → 5 parallel; 101–500 → 3 parallel; >500 → 2 parallel. Never spawn hundreds of subagents at once.
14. **Subagents report raw findings only.** Subagents do not recommend MERGE/ASK/SKIP and do not run decision logic. The parent's DECIDE scene is the only decision-maker.
15. **Identical subagent prompt.** Every assessment subagent uses `resources/subagent-prompt.md` verbatim with only `<number>`, `<repo_path>`, `<provider>`, and `<high_risk_globs>` substituted.
16. **State file is the only local write.** No PR-side or branch-side writes beyond the merge itself.
17. **General rate-limit → stop.** On a 429 from any non-merge API call, finish the in-flight batch, save state, exit. Do not retry within the same run. (A 429 specifically on the merge route is handled by the Merge Retry Policy in guardrail #12, not this one.)
18. **Resume is opt-in.** If `last_run` is older than 1 hour, ask the user before resuming. Never silently clear `processed`.

## References
- How to run (outer loops): `../_shared/runtime/gardener-running.md`
- Subagent assessment prompt: `resources/subagent-prompt.md`
- Provider command mappings: `../_shared/runtime/providers.md` (stub at `resources/providers.md`)
- Repository-specific risk globs: `resources/repo-rules.yaml`
- Execution protocol (state, batching, rate limits, drafts, verification): `resources/execution-protocol.md`
- Sibling skills: `gardener-sow` (create PRs), `gardener-tend` (maintain PRs)
- Context loading: `../_shared/core/context-loading.md`
- Reasoning templates: `../_shared/core/reasoning-templates.md`