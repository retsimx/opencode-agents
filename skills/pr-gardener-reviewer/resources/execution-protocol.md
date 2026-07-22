# Execution Protocol

Runtime behaviour for the pr-gardener-reviewer skill. The skill is pure
read-only against PRs (no push, no rebase, no CI fixing). This document covers
state, batching, parallelism, rate limits, drafts, large diffs, and branch
verification.

**Forge CLI commands:** use
[`../../_shared/runtime/providers.md`](../../_shared/runtime/providers.md)
(also stubbed at `resources/providers.md`). Do not invent provider-specific
commands outside that map.

## State file

**Path:** `.agents/results/pr-merge-queue.json`

**Schema:**

```json
{
  "last_run": "2026-07-18T09:00:00Z",
  "provider": "github",
  "processed": [
    {"number": 1678, "outcome": "merged"},
    {"number": 1677, "outcome": "skipped", "reason": "high_risk"},
    {"number": 1676, "outcome": "asked", "user_decision": "skip"},
    {"number": 1675, "outcome": "skipped", "reason": "draft"},
    {"number": 1674, "outcome": "skipped", "reason": "ci_pending"},
    {"number": 1673, "outcome": "skipped", "reason": "merge_failed"}
  ]
}
```

**Behaviour:**
- On Entry: load state file (create the directory and empty file if missing).
- Skip any PR whose number already appears in `processed`.
- After each PR decision, append to `processed` and save the file.
- At FINALIZE, leave the file on disk. The user can archive it manually between runs (or move it aside to re-process every PR).
- The state file is the only persistent local mutation the skill performs. PRs are never mutated.

## Scale: fetching all open PRs

- **GitHub**: `gh pr list --state open --json number,title,headRefName,createdAt,isDraft --limit 1000`. The `--limit` flag MUST be explicit — the default is 30 and silently under-fetches.
  - Above 1000 open PRs: `--limit 5000`, or paginate via `gh api 'search/issues?q=is:pr+is:open+repo:<owner>/<repo>&sort=created&order=asc&per_page=100&page=N'` until a page returns 0 items.
- **GitLab**: `glab mr list --state opened --output json --per-page 100 --page N`, page until a request returns 0 results.
- After fetch, always re-sort client-side by `createdAt` ascending so the oldest PR is processed first.
- Then filter drafts (`isDraft == false` on GitHub; `draft == false || work_in_progress == false` on GitLab) and any PR number already in the state file's `processed` list.
- Log the total count of remaining PRs to process before starting the first batch. This is the user's main signal of scope.

## Parallel assessment batching

- Default batch size: 5 subagent assessments spawned in parallel.
- **Scale rules — DO NOT spawn hundreds of subagents at once:**
  - 1–20 PRs total: batch size 5
  - 21–100 PRs total: batch size 5
  - 101–500 PRs total: batch size 3
  - > 500 PRs total: batch size 2
- Apply the merge gate sequentially in oldest-first order once a batch reports back.
- After merging a PR, the next batch starts. Never merge out of creation-date order even when reports arrive out of order.
- If a subagent task fails or times out, treat that PR as `skipped: subagent_failed` — do not block the rest of the batch, do not retry, do not investigate.

## Batched ASK pattern

- Accumulate all ASK-classified PRs across the current pass.
- After all assessment batches for the pass finish, present the accumulated ASKs in ONE `question` tool call (with `multiple: true`) letting the user multi-select which to proceed with.
- Chunk the question into batches of **≤ 10 PRs per call**. If more than 10 ASKs accumulate, issue successive `question` calls of 10 each until all have been presented. Do not present a single giant question.
- Each option label is `#<n> <title>` (1–5 words). The option description includes the subagent report's risk + behavior change + confidence + top concern summary.
- Merge each selected PR in turn (oldest first); record each into the state file.
- Unselected ASKs are recorded as `skipped` with `reason: "user_skipped_ask"`.
- If the user dismisses or cancels the question, record every un-answered ASK as `skipped: user_skipped_ask` and continue.

> Note: with high PR counts (300+), accumulating all ASKs across the whole pass before asking the user may leave the user waiting hours between question calls. Acceptable trade-off for safety. Do not attempt to mid-pass ask — that re-orders merges and risks state corruption.

## Rate-limit awareness

- Before listing PRs: check `gh api rate_limit` (GitHub) or `glab api rate_limit` (GitLab).
- If remaining core allowance < 200, set batch size to 1 and proceed slowly.
- On a 429 or rate-limit response from any API call: stop spawning new subagents immediately. Finish any pending batch, write state, exit with reason "rate_limited".

## Large-diff handling

- If a PR diff exceeds 50,000 chars (~10k tokens), record it as ASK with `reason: "large_diff"` before spawning a subagent.
- Never attempt to summarize or truncate diffs in-context; auto-ASK is the only safe response.
- This check must run in Entry or LIST, not in the subagent.

## Draft PR filter

- For BOTH providers: filter drafts client-side before any processing.
  - GitHub: `isDraft == false` (the `gh pr list --state open` output INCLUDES drafts by default — do not trust arguments to the contrary).
  - GitLab: `draft == false` and `work_in_progress == false`.
- Skill MUST skip all draft PRs with `skipped: draft` before assessing them. Never spawn an assessment subagent for a draft PR.

## CI status handling

- Pending CI is NOT a wait condition. If any check is `PENDING` or `RUNNING`, the PR is skipped with `reason: "ci_pending"`. Do not poll, do not sleep, do not retry. Move to the next PR.
- Failed checks: PR is skipped with `reason: "ci_failed"`. Do not investigate, do not run the failing tests, do not push fixes.
- All checks must be `PASS`/`SUCCESS` for auto-merge eligibility. Optional checks count as much as required checks: Y > 0 → SKIP. There is no "required vs optional" carve-out in this skill; if your repo uses optional checks, exclude them from CI at the provider level before relying on this skill.
- Zero checks observed (X = Y = Z = 0) → ASK with `reason: ci_unknown`. Never treat "no CI ran" as a clean run.

## Branch deletion verification

After a successful `gh pr merge N --squash --delete-branch` (or `glab mr merge N --squash --remove-source-branch`):

1. Fetch PR state via `gh pr view N --json state` (or glab equivalent). Expect `state == "MERGED"`.
   - If state is NOT merged, consult the Merge Retry Policy below before concluding anything.
2. Verify the branch ref is gone:
   - GitHub: `gh api repos/{owner}/{repo}/branches/{branch}` — expect 404.
   - GitLab: `glab api projects/:pid/repository/branches/{branch}` — expect 404.
3. Outcomes:
   - 404 → record `outcome: "merged"`.
   - 200 → record `outcome: "merged_branch_not_deleted"` and CONTINUE. Do not attempt to delete the branch manually — that is a mutation, and PR mutations are forbidden by the prime guardrail.
   - 403 or 5xx → trust `--delete-branch` succeeded; record `outcome: "merged"`.

## Merge execution: serial, with inter-merge gap

Merges MUST be issued serially (one `gh pr merge` / `glab mr merge` call at a time), never in parallel. GitHub and GitLab both rate-limit the merge endpoint and race internally with branch-protection rule re-evaluation. Issuing 5 merges near-simultaneously regularly produces transient 405 / 409 / 5xx failures on 2–3 of them even when the underlying PRs are clean.

Between merges, sleep 2 seconds (`sleep 2`) to give the provider's branch-protection state machine time to settle. This is a fixed, modest gap — not an indefinite poll.

## Merge Retry Policy

**A merge retry is NOT a PR mutation in the prime-invariant sense.** The user already authorized merging via this skill. Retrying the SAME squash-merge command (no code edits, no rebase, no force-push) is fully within that authorization.

When the merge command fails or `state != MERGED` after the first attempt, classify the failure:

**Transient — RETRY with backoff (up to 3 attempts):**
- HTTP 405 Method Not Allowed on the merge route
- HTTP 409 Conflict with message indicating "base branch policy is still being evaluated" or similar
- HTTP 429 rate-limited
- HTTP 5xx from the provider
- CLI exit code non-zero but stderr contains "merge queue", "temporarily", "try again", "Service Unavailable"
- `gh pr view N --json state` returns `OPEN` right after a non-error exit but pull request was actually merged (eventual-consistency lag)

**Permanent — SKIP, record, move on (NEVER retry):**
- HTTP 422 Unprocessable Entity — base branch conflict, NOT_MERGEABLE
- Branch protection rejects `--squash` → record `skipped: merge_strategy_mismatch`
- Branch protection rejects merge of any kind → record `skipped: branch_protection_blocked`
- PR has been closed/deleted out from under us → record `skipped: pr_disappeared`
- PR has new commits pushed since assessment (sha changed) → record `skipped: pr_changed_after_assessment`. The subagent's report is stale; do not re-assess in the same run.
- Any explicit conflict/diff mismatch message → record `skipped: merge_conflict`

**Backoff schedule (transient failures only):**
- Attempt 1: immediate (already done)
- Attempt 2: `sleep 5`, retry
- Attempt 3: `sleep 15`, retry
- After attempt 3 fails: record `skipped: merge_transient_failed` and continue to next PR

**Mandatory between retries:**
- Re-check `gh pr view N --json state` before each retry. If state is already `MERGED`, the previous failure was a false negative — record `outcome: "merged"` and move on. Do NOT issue another merge.
- Re-check `gh pr view N --json headRefOid` before each retry. If the commit SHA differs from the one captured at ASSESS time, abort retries — record `skipped: pr_changed_after_assessment`. The PR was mutated externally; do not trust the stale subagent report.

**Never do, even on retry:**
- Pull, fetch, rebase, push, force-push, commit, amend, edit any file, run any local test, run CI locally, delete a branch manually, investigate CI failure logs, change the merge strategy, change `--squash` flag, change `--delete-branch` flag. Any of these are violations of the prime invariant, regardless of retry motivation.

## Resume behaviour

- If the state file's `last_run` is recent (< 1 hour) AND more than 0 PRs are listed as `processed`, log a short resume banner and skip already-processed PRs.
- If `last_run` is older than 1 hour, ask the user via `question` whether to start fresh (clear `processed`) or resume.
- Never silently clear state — user confirmation required.