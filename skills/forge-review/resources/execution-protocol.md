# Forge Review: Technical Execution Protocol & Runbook

This document defines the complete technical runbook for orchestrating automated PR/MR reviews using the `forge-review` skill across GitHub and GitLab platforms.

---

## 1. Architecture Overview & Lifecycle

`forge-review` follows the **Universal File-First State I/O** and **Zero-Context Relay** architecture:
1. **Orchestrator** retrieves PR/MR metadata, diffs, and issue specifications.
2. Diffs and specifications are persisted to disk in `.agents/results/review-inputs-{sessionId}/`.
3. Three specialized subagents are dispatched concurrently with input file references (preventing conversational context saturation).
4. Subagents write structured analysis artifacts to disk and return standard 4-line summaries.
5. Orchestrator aggregates findings into the top-level review template (`review-template.md`) and formats inline suggestions (`comment-template.md`).
6. Orchestrator submits review and comments to the target forge via CLI / API.

```
┌─────────────────────────────────────────────────────────────┐
│                 Phase 1: Ingestion & Setup                  │
│  - Detect Forge Provider (GitHub / GitLab)                  │
│  - Fetch PR/MR Metadata, Diff, & Issue/Epic Specs           │
│  - Write: DIFF_FILE, SPEC_FILE, METADATA_FILE               │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│             Phase 2: Parallel Specialist Sweep              │
│  invoke_subagent([qa-agent, deep-reviewer, security-agent]) │
└──────┬───────────────────────┼───────────────────────┬──────┘
       │                       │                       │
       ▼                       ▼                       ▼
┌──────────────┐       ┌───────────────┐       ┌───────────────┐
│   QA Agent   │       │ Deep-Reviewer │       │Security Agent │
│ (AC Matrix & │       │ (9-Dimension  │       │ (Vulnerability│
│  Contracts)  │       │  Code Audit)  │       │  & Auth/IDOR) │
└──────┬───────┘       └───────┬───────┘       └───────┬───────┘
       │                       │                       │
       ▼                       ▼                       ▼
┌─────────────────────────────────────────────────────────────┐
│            Phase 3: Aggregation & Synthesis                 │
│  - Read all subagent result files from disk                 │
│  - Deduplicate findings & determine Verdict                 │
│  - Format Top-Level Review Body & Inline Diff Comments      │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│         Phase 4: Provider Review Submission & Gating        │
│  - Submit Top-Level Review & Inline Diff Comments via API   │
│  - Handle Rate Limits, Pagination, and Error Fallbacks      │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Provider Command Specifications

### A. GitHub Integration (`gh` CLI & REST API)

#### 1. Metadata Ingestion
```bash
# Fetch PR metadata (title, body, base branch, head commit, state)
gh pr view <PR_NUMBER> --json number,title,body,baseRefName,headRefOid,changedFiles,author,state > .agents/results/review-inputs-${SESSION_ID}/metadata.json

# Fetch PR full diff
gh pr diff <PR_NUMBER> > .agents/results/review-inputs-${SESSION_ID}/diff.patch

# Fetch associated issue or epic referenced in PR description (if applicable)
gh issue view <ISSUE_NUMBER> --json title,body,labels,assignees > .agents/results/review-inputs-${SESSION_ID}/spec-issue.json
```

#### 2. Review Submission via REST API (Atomic Batch Submission)
Submit top-level summary and all inline diff comments atomically in a single REST call:

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

> **Note on Multi-line Comments in GitHub API**:
> - Single-line comment: `"line": <line_number>`, `"side": "RIGHT"`
> - Multi-line comment: `"start_line": <start_line_number>`, `"line": <end_line_number>`, `"start_side": "RIGHT"`, `"side": "RIGHT"`

#### 3. Single Comment Fallback (If batch review payload fails)
```bash
# Add a single review comment on diff
gh api \
  --method POST \
  -H "Accept: application/vnd.github+json" \
  /repos/{owner}/{repo}/pulls/{pull_number}/comments \
  -f body="[BUG] Explanation..." \
  -f commit_id="{HEAD_SHA}" \
  -f path="tutoring/views.py" \
  -F line=88 \
  -f side="RIGHT"
```

---

### B. GitLab Integration (`glab` CLI & REST API)

#### 1. Metadata Ingestion
```bash
# Fetch MR metadata
glab mr view <MR_IID> --output json > .agents/results/review-inputs-${SESSION_ID}/metadata.json

# Fetch MR full diff
glab mr diff <MR_IID> > .agents/results/review-inputs-${SESSION_ID}/diff.patch

# Fetch MR commit SHA information
glab api /projects/:id/merge_requests/<MR_IID>/versions > .agents/results/review-inputs-${SESSION_ID}/versions.json
```

#### 2. Inline Discussion Submission via REST API
GitLab creates inline comments via discussions with position coordinates:

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

## 3. Subagent Prompt Templates (Zero-Context Relay)

When the orchestrator triggers Phase 2, it launches the 3 subagents concurrently via `invoke_subagent`. All heavy inputs (`DIFF_FILE`, `SPEC_FILE`, `METADATA_FILE`) are passed **by file reference on disk**, never dumped as large raw strings into the prompt.

### Prompt Template 1: `qa-agent` (Contract & Acceptance Criteria Specialist)

```json
{
  "TypeName": "self",
  "Role": "QA Contract Specialist",
  "Prompt": "You are the QA Contract Specialist for forge-review.\n\n### Task Context:\n- SESSION_ID: \"{SESSION_ID}\"\n- DIFF_FILE: \"{DIFF_FILE_PATH}\"\n- SPEC_FILE: \"{SPEC_FILE_PATH}\"\n- METADATA_FILE: \"{METADATA_FILE_PATH}\"\n- OUTPUT_FILE: \".agents/results/result-qa-{SESSION_ID}.md\"\n\n### Objective:\nVerify 100% of Acceptance Criteria and contract requirements from SPEC_FILE against the code diff in DIFF_FILE.\n\n### Instructions:\n1. Read SPEC_FILE and extract every functional requirement, edge case, and acceptance criterion.\n2. Read DIFF_FILE and investigate touched files in the workspace.\n3. Build the Acceptance Criteria & Contract Alignment Matrix with explicit `file:line` proof citations for every item.\n4. Determine status: `VERIFIED`, `INCOMPLETE`, `DEVIATED`, or `MISSING`.\n5. Write complete analysis to OUTPUT_FILE.\n6. Return strictly the 4-line chat completion summary."
}
```

---

### Prompt Template 2: `deep-reviewer` (9-Dimension Code Quality Specialist)

```json
{
  "TypeName": "self",
  "Role": "Deep Review Specialist",
  "Prompt": "You are the Deep Review Specialist for forge-review.\n\n### Task Context:\n- SESSION_ID: \"{SESSION_ID}\"\n- DIFF_FILE: \"{DIFF_FILE_PATH}\"\n- OUTPUT_FILE: \".agents/results/result-deep-review-{SESSION_ID}.md\"\n\n### Objective:\nPerform a deterministic, evidence-based 9-dimension code audit on the changes in DIFF_FILE.\n\n### Evaluation Dimensions:\n1. Correctness (logic errors, broken invariants, unhandled conditions)\n2. Regression Risk (broken existing workflows, contract drift)\n3. State & Data Integrity (DB migrations, transaction boundaries, concurrency)\n4. UI / Rendering / UX (template tags, CSS/HTMX states, error handling)\n5. Test Coverage & Quality (missing assertions, test gaps, deterministic mocks)\n6. Performance & Scalability (N+1 queries, memory bottlenecks, unindexed lookups)\n7. Dead Code & Hygiene (unreachable code, unused imports)\n8. DRY & Architectural Consistency (duplicated logic, pattern conformity)\n9. Code Style & Maintainability (naming clarity, docstring accuracy)\n\n### Instructions:\n1. Read DIFF_FILE and trace execution paths through modified files.\n2. Identify defects and draft inline ` ```suggestion ` blocks following .agents/skills/forge-review/resources/comment-template.md.\n3. Write full structured review report to OUTPUT_FILE.\n4. Return strictly the 4-line chat completion summary."
}
```

---

### Prompt Template 3: `security-agent` (Security & Threat Specialist)

```json
{
  "TypeName": "self",
  "Role": "Security Review Specialist",
  "Prompt": "You are the Security Review Specialist for forge-review.\n\n### Task Context:\n- SESSION_ID: \"{SESSION_ID}\"\n- DIFF_FILE: \"{DIFF_FILE_PATH}\"\n- OUTPUT_FILE: \".agents/results/result-security-{SESSION_ID}.md\"\n\n### Objective:\nPerform an exhaustive security and vulnerability audit on DIFF_FILE.\n\n### Threat Audit Dimensions:\n1. Authentication & Session Management (token handling, session fixation)\n2. Authorization & IDOR (object-level permissions, tenant scoping)\n3. Injection Flaws (SQLi, Command Injection, Template Injection, XSS)\n4. CSRF & State Mutation Protection (CSRF tokens, method constraints)\n5. Sensitive Data Exposure (leaked secrets, unmasked PII, insecure logging)\n6. Insecure Dependencies & Cryptographic Misconfigurations\n\n### Instructions:\n1. Read DIFF_FILE and search for authorization gaps and input sanitization flaws.\n2. Formulate concrete exploit scenarios for any surfaced vulnerabilities.\n3. Formulate precise remediation suggestion blocks following .agents/skills/forge-review/resources/comment-template.md.\n4. Write complete security audit artifact to OUTPUT_FILE.\n5. Return strictly the 4-line chat completion summary."
}
```

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

#### GitHub Rate Limit Headers:
- `x-ratelimit-remaining`: Check remaining request budget.
- `x-ratelimit-reset`: Unix epoch timestamp when the primary quota resets.
- `retry-after`: Seconds to wait (if returned during secondary rate limit abuse detection).

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
3. If partial findings exist, ingest them with a `[PARTIAL_EVALUATION]` annotation.
4. Record an explicit audit note in the top-level review:
   `> [!WARNING] The Security Specialist evaluation was partially degraded due to tool timeouts. Manual review of authentication boundaries recommended.`
5. Set review verdict to `COMMENT` or `REQUEST_CHANGES` depending on the severity of available findings.
