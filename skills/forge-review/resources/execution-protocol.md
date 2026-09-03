# Forge Review: Technical Execution Protocol & Runbook

This document defines the authoritative technical execution protocol and operational runbook for orchestrating automated PR/MR reviews using the `forge-review` skill across GitHub (`gh`) and GitLab (`glab`) platforms.

---

## 1. Architecture Overview & Lifecycle (5-Agent Topology)

`forge-review` operates under **Universal File-First State I/O**, **Zero-Context Relay**, and **Isolated Zero-Trust Security**:

1. **Subagent 0 (`context-ingestion`)**: Interacts with the target forge CLI (`gh` or `glab`), extracts PR/MR metadata, diffs, and issue specifications, prunes bot noise, excludes lockfiles/minified assets, sanitizes untrusted markdown metadata with entity-encoding and dynamic session nonces, guarantees raw uncorrupted code diff syntax in `diff-pr.patch` wrapped inside `<untrusted_diff session_nonce="...">`, normalizes author roles, and persists unconstrained artifacts to disk with **ZERO token limits**. It separates **current-state** (`pr-context.md`: metadata, head SHA, author intent, documented deviations, sanitized description) from **historical review findings** (`pr-history.md`: prior-round findings under a clearly-marked "HISTORICAL REVIEW ROUNDS (may describe already-fixed code)" section), tagging provenance at ingestion without determining staleness.
2. **Phase 2 (Parallel Detector Sweep)**: Orchestrator concurrently dispatches three domain specialists:
   - **Subagent 1 (`qa-agent`)**: Reads `spec-issue.md`, `pr-context.md`, `diff-pr.patch`, and `.agents/skills/review/SKILL.md` to evaluate 100% of Acceptance Criteria and contract commitments for Section 1 (Acceptance Criteria & Contract Alignment Matrix with `file:line` proof citations).
   - **Subagent 2 (`deep-reviewer`)**: Reads `pr-context.md`, `diff-pr.patch`, `.agents/skills/deep-review/SKILL.md`, and `docs/checklists/{domain}.md` to perform an exhaustive 9-dimension code audit for Section 3 (9-Dimension Quality Scorecard) and stage candidate diff suggestions with ` ```suggestion ` blocks.
   - **Subagent 3 (`security-agent`)**: Reads `diff-pr.patch` **ONLY** under an isolated **STRICT ZERO-TRUST MANDATE** (completely barred from author explanations, narrative justifications, or issue descriptions) to generate Section 2 Dedicated Threat Model Matrix across 6 threat vectors with exploit scenarios, reachability paths, and remediation code.
3. **Phase 3 (Intermediate Synthesis)**: Orchestrator aggregates specialist outputs into an intermediate raw synthesis (`.agents/results/raw-findings-pr-{n}-{sessionId}.md`).
4. **Phase 3.5 (Verification & Criticism Pass)**: Orchestrator dispatches **Subagent 4 (`review-verifier`)** to execute the **5-Check Verification Protocol** against live repository source files and diff hunks, enforcing the **provenance gate** (every finding must cite a current-head `file:line`; no citation → demote to Section 5 or drop) and the **Immutable Security Pass-Through Invariant**, formatting Section 4 (Staged Inline Diff Suggestions with Badge + Location + Problem + Remediation + ` ```suggestion ` blocks) and Section 5 (Out-of-Diff Observations), and emitting the complete 6-section master deliverable (`.agents/results/forge-review/<sessionId>/review-pr-{n}-verified.md` or `.agents/results/review-pr-{n}-{sessionId}.md`). If `pr-history.md` is provided, Subagent 4 MAY cross-reference prior-round findings to note "previously raised, now verified fixed" as an optional courtesy.
5. **Phase 4 (Presentation, Gate, & Publication)**:
   - **Scene 5 (Orchestrator Presentation & Human Gate)**: Orchestrator reads `.agents/results/forge-review/<sessionId>/review-pr-{n}-verified.md` (or `.agents/results/review-pr-{n}-{sessionId}.md`) and prints its **complete, untruncated, uncollapsed markdown contents directly to the chat window** (including all rich tables, `file:line` proof citations, exploit scenarios, and ` ```suggestion ` replacement blocks) before asking the user. Replacing rich tables or suggestion blocks with summarized one-liners or file references is strictly forbidden. The orchestrator halts at the mandatory Human Approval Gate (ask the user).
   - **Scene 6 (Forge Publication)**: Upon explicit human approval, submits verified atomic batch reviews and inline diff comments via provider REST API payloads (`gh api` or `glab api`).

```
┌────────────────────────────────────────────────────────────────────────────┐
│              Phase 1: Ingestion & Sanitization Specialist                  │
│  spawn a subagent([context-ingestion])                                     │
│  - Query Forge CLI (gh / glab) for PR/MR, Diff, & Issue Specs              │
│  - Prune Bot Accounts (*[bot], codecov, github-actions, dependabot)        │
│  - Strip CI Tables, Badges, HTML Comments, and Redundant Logs              │
│  - Exclude Lockfiles (*.lock, pnpm-lock.yaml) & Minified Assets (*.min.*)  │
│  - Entity-Encode Text Metadata (<, >) & Wrap Nonce Boundaries              │
│  - Preserve Raw Diff Syntax Wrapped in <untrusted_diff session_nonce="...">│
│  - Normalize Maintainer Roles (MAINTAINER / CONTRIBUTOR / EXTERNAL)        │
│  - Separate current-state (pr-context.md) from historical findings         │
│    (pr-history.md); tag provenance, do not determine staleness             │
│  - Write: spec-issue.md, pr-context.md, pr-history.md, diff-pr.patch       │
│    (ZERO Token Limits)                                                     │
└────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                    Phase 2: Parallel Specialist Sweep                      │
│  spawn a subagent([qa-agent, deep-reviewer, security-agent])               │
└────────────────────────────────────────────────────────────────────────────┘
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
│    & Contract Alignment  │ │    Code Quality   │ │  - Threat Model Matrix   │
│    (Section 1)           │ │    (Section 3)    │ │    (6 Vectors) (Section 2│
└─────────────┬────────────┘ └────────┬──────────┘ └──────────┬───────────────┘
              │                       │                       │
              └───────────────────────┼───────────────────────┘
                                      ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                     Phase 3: Intermediate Synthesis                       │
│  - Orchestrator collects Subagents 1, 2, 3 result files from disk         │
│  - Aggregates raw findings into raw-findings-pr-{n}-{sessionId}.md        │
└───────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────────────────┐
│             Phase 3.5: Review Verifier & Critic Specialist Pass            │
│  spawn a subagent([review-verifier])                                       │
│  - Check 1: Ground Truth Fact-Check (Verify against live codebase)         │
│    + Provenance Gate: every finding MUST cite a current-head file:line;    │
│      no citation -> demote to Section 5 or drop                            │
│  - Check 2: Diff Hunk Line Bounds & 422 Demotion (Validate hunk spans)     │
│  - Check 3: Suggestion Syntax & Indentation Normalization                  │
│  - Check 4: Cross-Specialist Deduplication & Severity Recalibration        │
│  - Check 5: Immutable Security Pass-Through Invariant (Subagent 3 Locked)  │
│  - Format Section 4 Staged Suggestions & Section 5 Out-of-Diff Obs         │
│  - Write: OUTPUT_FILE (.agents/results/review-pr-{n}-{sessionId}.md or     │
│    .agents/results/forge-review/<sessionId>/review-pr-{n}-verified.md)     │
│    (Complete 6-Section Master Deliverable)                                 │
└────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│           Phase 4: Scene 5 Presentation Gate & Scene 6 Publication          │
│  - Scene 5: Print Complete Uncollapsed 6-Section Markdown to Chat Window    │
│    (Strict prohibition on one-liner summaries or file reference links)      │
│  - Scene 5: Enforce Mandatory Human Approval Gate by asking the user        │
│  - Scene 6: Submit Atomic Batch Review Payload & Inline Diff Comments API   │
│  - Scene 6: Execute Rate-Limit Exponential Backoff & 422 Outdated Fallback  │
└─────────────────────────────────────────────────────────────────────────────┘
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
GitLab posts inline comments as discussion threads using exact diff version coordinate hashes. Both single-line and multi-line discussion schemas are supported:

```bash
# Extract base_sha, start_sha, and head_sha from latest MR version
BASE_SHA=$(jq -r '.[0].base_commit_sha' .agents/results/review-inputs-${SESSION_ID}/versions.json)
START_SHA=$(jq -r '.[0].start_commit_sha' .agents/results/review-inputs-${SESSION_ID}/versions.json)
HEAD_SHA=$(jq -r '.[0].head_commit_sha' .agents/results/review-inputs-${SESSION_ID}/versions.json)

# A. Single-Line Discussion Submission
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

# B. Multiline Discussion Submission (line_range schema)
glab api \
  --method POST \
  /projects/:id/merge_requests/<MR_IID>/discussions \
  -f body="**[BUG]**: Handle None return from get_active_term().\n\n```suggestion\n    term = get_active_term(target_date)\n    if not term:\n        return []\n    max_slots = term.max_daily_sessions\n```" \
  -f "position[position_type]=text" \
  -f "position[base_sha]=${BASE_SHA}" \
  -f "position[start_sha]=${START_SHA}" \
  -f "position[head_sha]=${HEAD_SHA}" \
  -f "position[new_path]=tutoring/views.py" \
  -F "position[line_range][start][new_line]=42" \
  -f "position[line_range][start][type]=new" \
  -F "position[line_range][end][new_line]=45" \
  -f "position[line_range][end][type]=new"
```

> **GitLab Diff Positioning Rules**:
> - Single-line comment: `"position[new_line]": <line_number>` (integer via `-F`).
> - Multiline comment: `"position[line_range][start][new_line]": <start_line>`, `"position[line_range][start][type]": "new"`, `"position[line_range][end][new_line]": <end_line>`, `"position[line_range][end][type]": "new"`.
> - Base, Start, and Head SHAs MUST match the MR version metadata to avoid HTTP 400/422 errors.

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

## 3. Subagent Prompt Contracts & Operational Runbooks

### A. Subagent Prompt Contracts (Mandatory Full Rich Markdown)
All subagents are dispatched via the subagent tool using the prompt templates defined in `.agents/skills/forge-review/resources/subagent-prompts.md`. All prompts strictly mandate producing complete, uncollapsed rich markdown deliverables without placeholders or abbreviated one-liners:

- **Subagent 0 (`context-ingestion`)**:
  - Ingests and sanitizes metadata, issue specs, and unified diffs with **ZERO token limits**.
  - Generates `spec-issue.md`, `pr-context.md`, and `diff-pr.patch` with dynamic nonces and raw uncorrupted code diff syntax.
- **Subagent 1 (`qa-agent`)**:
  - Validates 100% of Acceptance Criteria and contract commitments against the code diff.
  - Produces complete Section 1 Acceptance Criteria & Contract Alignment Matrix with status badges (`VERIFIED`, `INCOMPLETE`, `DEVIATED`, `MISSING`), exact `file:line` proof citations, and verification details.
- **Subagent 2 (`deep-reviewer`)**:
  - Audits diff across all 9 code quality dimensions.
  - Produces Section 3 9-Dimension Quality Scorecard table and detailed findings breakdowns per dimension with concrete runtime execution traces and drafted ` ```suggestion ` replacement blocks.
- **Subagent 3 (`security-agent`)**:
  - Operates under isolated Strict Zero-Trust (diffs only, barred from narrative justifications).
  - Produces Section 2 Dedicated Threat Model Matrix across all 6 threat vectors with full Exploit Scenarios, reachability paths, impact analysis, and remediation suggestion blocks.
- **Subagent 4 (`review-verifier`)**:
  - Executes the 5-Check Verification Protocol (Fact-checking, Diff Hunk Bounds & 422 Demotion, Suggestion Syntax & Indentation Normalization, Deduplication & Severity Recalibration, Immutable Security Pass-Through).
  - Synthesizes and emits the final verified deliverable (`.agents/results/forge-review/<sessionId>/review-pr-{n}-verified.md` or `.agents/results/review-pr-{n}-{sessionId}.md`) formatted with all 6 rich markdown sections fully populated.

### B. Scene 5: Orchestrator Presentation & Human Gate Protocol
Prior to triggering the interactive human gate or publishing to any forge:
1. **Mandatory Uncollapsed Presentation**:
   - The Orchestrator MUST read the verified review artifact (`.agents/results/forge-review/<sessionId>/review-pr-{n}-verified.md` or `.agents/results/review-pr-{n}-{sessionId}.md`) from disk.
   - The Orchestrator MUST print the **complete, untruncated, uncollapsed markdown contents directly to the active chat window** before asking the user.
   - **Strict Prohibition**: The Orchestrator is **STRICTLY FORBIDDEN** from replacing rich markdown tables or ` ```suggestion ` blocks with summarized one-liners, bulleted digests, or mere file links / references.
   - Every table row, status badge, code proof citation, exploit trace, problem explanation, and ` ```suggestion ` code block must be fully rendered in chat.
2. **Interactive Human Approval Gate**:
   - The Orchestrator presents interactive options by asking the user (Publish Review + Comments, Summary Only, Revise, Abort).
   - **Hard Barrier**: Never publish or mutate remote forge state without explicit user confirmation.

### C. Operational Runbooks Reference
See `.agents/skills/forge-review/resources/operational-runbooks.md` for detailed runbook specifications:
- **Section 1**: Subagent 0 Operational Runbook: Context Ingestion & Sanitization
- **Section 2**: Subagent 4 Operational Runbook: Review Verifier & Critic Specialist (5-Check Protocol)
- **Section 3**: Forge Discussion Payload Reference: GitLab Multiline Discussion Schema
- **Section 4**: GitHub Atomic Batch Review Payload Reference & Submission Protocol
- **Section 5**: Scene 5: Orchestrator Presentation & Human Gate Runbook

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
   - **Step 3 (Graceful Fallback)**: If re-anchoring fails, DO NOT fail the review. Downgrade the unplaced inline comment to a labeled finding inside **Section 5 (Out-of-Diff Observations)** of the top-level review body (`review-template.md`), citing the target `file:line` directly in markdown text.

---

### C. Review Body Size Limit & Truncation

GitHub imposes a 65,536-character limit on review comments and issue bodies.
1. **Detection**: Measure character count of synthesized `review-summary-${SESSION_ID}.md`.
2. **Handling**:
   - If length > 60,000 characters:
     - Retain full Acceptance Criteria Matrix (Section 1), Threat Model Matrix (Section 2), and 9-Dimension summary table (Section 3).
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
