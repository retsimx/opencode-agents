# Forge Review: Technical Execution Protocol & Runbook

This document defines the authoritative technical execution protocol and operational runbook for orchestrating automated PR/MR reviews using the `forge-review` skill across GitHub (`gh`) and GitLab (`glab`) platforms.

---

## 1. Architecture Overview & Lifecycle (5-Agent Topology)

`forge-review` operates under **Universal File-First State I/O**, **Zero-Context Relay**, and **Isolated Zero-Trust Security**:

1. **Subagent 0 (`context-ingestion`)**: Interacts with the target forge CLI (`gh` or `glab`), extracts PR/MR metadata, diffs, and issue specifications, prunes bot noise, excludes lockfiles/minified assets, sanitizes untrusted input with entity-encoding and dynamic session nonces, normalizes author roles, and persists unconstrained artifacts to disk with **ZERO token limits**.
2. **Phase 2 (Parallel Detector Sweep)**: Orchestrator concurrently dispatches three domain specialists:
   - **Subagent 1 (`qa-agent`)**: Reads `spec-issue.md`, `pr-context.md`, `diff-pr.patch`, and `.agents/skills/review/SKILL.md` to evaluate 100% of Acceptance Criteria and contract commitments.
   - **Subagent 2 (`deep-reviewer`)**: Reads `pr-context.md`, `diff-pr.patch`, `.agents/skills/deep-review/SKILL.md`, and `docs/checklists/{domain}.md` to perform an exhaustive 9-dimension code audit and stage line-level inline suggestions.
   - **Subagent 3 (`security-agent`)**: Reads `diff-pr.patch` **ONLY** under an isolated **STRICT ZERO-TRUST MANDATE** (completely barred from author explanations, narrative justifications, or issue descriptions).
3. **Phase 3 (Intermediate Synthesis)**: Orchestrator aggregates specialist outputs into an intermediate raw synthesis (`.agents/results/raw-findings-pr-{n}-{sessionId}.md`).
4. **Phase 3.5 (Verification & Criticism Pass)**: Orchestrator dispatches **Subagent 4 (`review-verifier`)** to execute the **5-Check Verification Protocol** against live repository source files and diff hunks, enforcing the **Immutable Security Pass-Through Invariant** and emitting the pristine review deliverable (`.agents/results/review-pr-{n}-{sessionId}.md`).
5. **Phase 4 (Presentation, Gate, & Publication)**: Orchestrator presents the verified scorecard in chat, enforces the Human Approval Gate via `ask_question`, and publishes verified batch reviews and inline diff comments via atomic REST API payloads.

```
┌───────────────────────────────────────────────────────────────────────────┐
│              Phase 1: Ingestion & Sanitization Specialist                 │
│  invoke_subagent([context-ingestion])                                     │
│  - Query Forge CLI (gh / glab) for PR/MR, Diff, & Issue Specs             │
│  - Prune Bot Accounts (*[bot], codecov, github-actions, dependabot)       │
│  - Strip CI Tables, Badges, HTML Comments, and Redundant Logs             │
│  - Exclude Lockfiles (*.lock, pnpm-lock.yaml) & Minified Assets (*.min.*) │
│  - Entity-Encode (<, >) & Wrap Content with Cryptographic Session Nonces  │
│  - Normalize Maintainer Roles (MAINTAINER / CONTRIBUTOR / EXTERNAL)       │
│  - Write: spec-issue.md, pr-context.md, diff-pr.patch (ZERO Token Limits) │
└─────────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                    Phase 2: Parallel Specialist Sweep                     │
│  invoke_subagent([qa-agent, deep-reviewer, security-agent])               │
└─────────────┬───────────────────────┼───────────────────────┬─────────────┘
              │                       │                       │
              ▼                       ▼                       ▼
┌──────────────────────────┐ ┌───────────────────┐ ┌──────────────────────────┐
│         Subagent 1       │ │    Subagent 2     │ │        Subagent 3        │
│          qa-agent        │ │   deep-reviewer   │ │      security-agent      │
│  - Reads: spec-issue.md, │ │  - Reads:         │ │  - Reads: diff-pr.patch  │
│    pr-context.md,        │ │    pr-context.md, │ │    ONLY                  │
│    diff-pr.patch,        │ │    diff-pr.patch, │ │  - STRICT ZERO-TRUST:    │
│    skills/review         │ │    deep-review    │ │    Barred from author    │
│  - Acceptance Criteria   │ │  - 9-Dimension    │ │    narrative or excuses  │
│    & Contract Alignment  │ │    Code Quality   │ │  - OWASP Top 10 & Auth   │
└─────────────┬────────────┘ └────────┬──────────┘ └──────────┬───────────────┘
              │                       │                       │
              └───────────────────────┼───────────────────────┘
                                      ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                     Phase 3: Intermediate Synthesis                       │
│  - Orchestrator collects Subagents 1, 2, 3 result files from disk         │
│  - Aggregates raw findings into raw-findings-pr-{n}-{sessionId}.md        │
└─────────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
┌───────────────────────────────────────────────────────────────────────────┐
│             Phase 3.5: Review Verifier & Critic Specialist Pass           │
│  invoke_subagent([review-verifier])                                       │
│  - Check 1: Ground Truth Fact-Check (Verify against live codebase)        │
│  - Check 2: Diff Hunk Line Bounds & 422 Demotion (Validate hunk spans)    │
│  - Check 3: Suggestion Syntax & Indentation Normalization                 │
│  - Check 4: Cross-Specialist Deduplication & Severity Recalibration       │
│  - Check 5: Immutable Security Pass-Through Invariant (Subagent 3 Locked) │
│  - Write: OUTPUT_FILE (.agents/results/review-pr-{n}-{sessionId}.md)      │
└─────────────────────────────────────┬─────────────────────────────────────┘
                                      │
                                      ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                Phase 4: Provider Review Submission & Gating               │
│  - Present Verified Scorecard & Gate via ask_question (Mandatory)         │
│  - Submit Atomic Batch Review Payload & Inline Diff Comments via REST API │
│  - Execute Rate-Limit Exponential Backoff & 422 Outdated Hunk Fallbacks   │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Provider Command Specifications

### A. GitHub Integration (`gh` CLI & REST API)

#### 1. Ingestion via `gh` CLI
```bash
# Fetch PR metadata (title, body, base branch, head commit, author, state)
gh pr view <PR_NUMBER> --json number,title,body,baseRefName,headRefOid,changedFiles,author,state > .agents/results/review-inputs-${SESSION_ID}/raw-metadata.json

# Fetch PR full unified diff
gh pr diff <PR_NUMBER> > .agents/results/review-inputs-${SESSION_ID}/raw-diff.patch

# Fetch associated issue or epic referenced in PR description (if applicable)
gh issue view <ISSUE_NUMBER> --json number,title,body,labels,assignees,author > .agents/results/review-inputs-${SESSION_ID}/raw-spec-issue.json
```

#### 2. Atomic Batch Review Submission via REST API
Submit top-level review summary and all inline diff comments atomically in a single REST call:

```bash
gh api \
  --method POST \
  -H "Accept: application/vnd.github+json" \
  /repos/{owner}/{repo}/pulls/{pull_number}/reviews \
  --input - << 'EOF'
{
  "commit_id": "{HEAD_SHA}",
  "body": "{TOP_LEVEL_REVIEW_MARKDOWN}",
  "event": "REQUEST_CHANGES",
  "comments": [
    {
      "path": "tutoring/views.py",
      "line": 88,
      "side": "RIGHT",
      "body": "**[BUG]**: Using `timezone.now().date()` causes timezone offset errors.\n\n```suggestion\n    today = timezone.localdate()\n```"
    },
    {
      "path": "tutoring/services/slot_calculator.py",
      "start_line": 42,
      "line": 45,
      "start_side": "RIGHT",
      "side": "RIGHT",
      "body": "**[BUG]**: Handle `None` return from `get_active_term()`.\n\n```suggestion\n    term = get_active_term(target_date)\n    if not term:\n        return []\n    max_slots = term.max_daily_sessions\n```"
    }
  ]
}
EOF
```

> **GitHub Diff Positioning Rules**:
> - Single-line comment: `"line": <line_number>`, `"side": "RIGHT"`
> - Multi-line comment: `"start_line": <start_line_number>`, `"line": <end_line_number>`, `"start_side": "RIGHT"`, `"side": "RIGHT"`
> - Target lines must fall strictly within the additions/context lines of the latest commit (`HEAD_SHA`).

#### 3. Single Comment Fallback (If batch review payload fails)
```bash
# Add an individual review comment on diff
gh api \
  --method POST \
  -H "Accept: application/vnd.github+json" \
  /repos/{owner}/{repo}/pulls/{pull_number}/comments \
  -f body="**[BUG]**: Local date fix required.\n\n```suggestion\n    today = timezone.localdate()\n```" \
  -f commit_id="{HEAD_SHA}" \
  -f path="tutoring/views.py" \
  -F line=88 \
  -f side="RIGHT"
```

---

### B. GitLab Integration (`glab` CLI & REST API)

#### 1. Ingestion via `glab` CLI
```bash
# Fetch MR metadata
glab mr view <MR_IID> --output json > .agents/results/review-inputs-${SESSION_ID}/raw-metadata.json

# Fetch MR full unified diff
glab mr diff <MR_IID> > .agents/results/review-inputs-${SESSION_ID}/raw-diff.patch

# Fetch MR commit SHA information (base_commit_sha, start_commit_sha, head_commit_sha)
glab api /projects/:id/merge_requests/<MR_IID>/versions > .agents/results/review-inputs-${SESSION_ID}/versions.json
```

#### 2. Inline Discussion Submission via REST API
GitLab posts inline comments as discussion threads using exact diff version coordinate hashes:

```bash
# Extract base_sha, start_sha, and head_sha from latest MR version
BASE_SHA=$(jq -r '.[0].base_commit_sha' .agents/results/review-inputs-${SESSION_ID}/versions.json)
START_SHA=$(jq -r '.[0].start_commit_sha' .agents/results/review-inputs-${SESSION_ID}/versions.json)
HEAD_SHA=$(jq -r '.[0].head_commit_sha' .agents/results/review-inputs-${SESSION_ID}/versions.json)

# Submit inline diff discussion
glab api \
  --method POST \
  /projects/:id/merge_requests/<MR_IID>/discussions \
  -f body="**[BUG]**: Local date fix required.\n\n```suggestion\n    today = timezone.localdate()\n```" \
  -f "position[position_type]=text" \
  -f "position[base_sha]=${BASE_SHA}" \
  -f "position[start_sha]=${START_SHA}" \
  -f "position[head_sha]=${HEAD_SHA}" \
  -f "position[new_path]=tutoring/views.py" \
  -F "position[new_line]=88"
```

#### 3. Top-Level Review Summary & Approval
```bash
# Post top-level summary note
glab mr note <MR_IID> --message "$(cat .agents/results/review-summary-${SESSION_ID}.md)"

# If verdict is APPROVE:
glab mr approve <MR_IID>

# If verdict is REQUEST_CHANGES (unapprove if previously approved):
glab mr unapprove <MR_IID>
```

---

## 3. Subagent Prompt Templates & Operational Runbooks

- **Subagent Prompt Templates**: See `.agents/skills/forge-review/resources/subagent-prompts.md` for JSON templates for Subagents 0, 1, 2, 3, and 4.
- **Operational Runbooks**: See `.agents/skills/forge-review/resources/operational-runbooks.md` for Subagent 0 Ingestion Runbook and Subagent 4 5-Check Verification Runbook.

---

## 4. Rate-Limit Backoff & Error Recovery Policies

### A. Rate-Limit Detection & Exponential Backoff

When executing API calls (`gh api` or `glab api`), automated reviews may encounter provider rate limits (HTTP 429 Too Many Requests or HTTP 403 Forbidden with rate limit headers).

#### Backoff Algorithm
```
WaitTime = min(MaxWait, InitialWait * (2 ^ Attempt) + RandomJitter)
- InitialWait = 2 seconds
- MaxWait = 60 seconds
- RandomJitter = uniform_random(0.5, 2.0) seconds
- MaxRetries = 4 attempts
```

#### Bash Execution Snippet with Backoff:
```bash
call_api_with_retry() {
  local retries=4
  local count=0
  local wait_sec=2
  until "$@"; do
    exit_code=$?
    count=$((count + 1))
    if [ $count -ge $retries ]; then
      echo "API call failed after $retries attempts: $@" >&2
      return $exit_code
    fi
    sleep_time=$((wait_sec * (2 ** (count - 1)) + (RANDOM % 3)))
    echo "Warning: API rate limit or error encountered. Retrying in ${sleep_time}s (Attempt $count/$retries)..." >&2
    sleep $sleep_time
  done
}
```

---

### B. Outdated Diff Hunk Recovery Policy

When submitting an inline suggestion to GitHub/GitLab:
1. **Cause**: If the PR author pushes a new commit during review, or if line numbering drifted, the API will return `422 Unprocessable Entity: Line is not part of the diff`.
2. **Recovery Procedure**:
   - **Step 1**: Attempt to re-fetch the latest commit SHA and diff.
   - **Step 2**: Re-anchor line numbers against the refreshed diff hunk.
   - **Step 3 (Graceful Fallback)**: If re-anchoring fails, DO NOT fail the review. Downgrade the unplaced inline comment to a labeled finding inside **Section 3.A (Blocking Findings)** or **Section 3.B** of the top-level review body (`review-template.md`), citing the target `file:line` directly in markdown text.

---

### C. Review Body Size Limit & Truncation

GitHub imposes a 65,536-character limit on review comments and issue bodies.
1. **Detection**: Measure character count of synthesized `review-summary-${SESSION_ID}.md`.
2. **Handling**:
   - If length > 60,000 characters:
     - Retain full Acceptance Criteria Matrix and 9-Dimension summary table.
     - Move extended stack traces or voluminous duplicate code snippets to an attached report artifact `.agents/results/result-forge-review-extended-${SESSION_ID}.md`.
     - Ensure the primary review comment remains clean, actionable, and strictly under the 65 KiB threshold.

---

### D. Subagent Failure / Degradation Policy

If any subagent encounters an error or returns `Status: FAILED`:
1. The orchestrator must not crash.
2. Inspect the failed subagent's `OUTPUT_FILE` diagnostic report.
3. If partial findings exist from Subagents 1, 2, or 3, ingest them into `RAW_REVIEW_FILE` with a `[PARTIAL_EVALUATION]` annotation before passing to `review-verifier`.
4. If Subagent 4 (`review-verifier`) encounters a failure, orchestrator falls back to raw synthesis with automated lint validation or retries Subagent 4.
5. Record an explicit audit note in the top-level review:
   `> [!WARNING] The Security Specialist evaluation was partially degraded due to tool timeouts. Manual review of authentication boundaries recommended.`
6. Set review verdict to `COMMENT` or `REQUEST_CHANGES` depending on the severity of available findings.
