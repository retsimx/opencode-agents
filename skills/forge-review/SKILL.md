---
name: forge-review
description: >
  Autonomous, maximum-depth alignment audit, 9-dimension deep review, and security audit of
  PRs/MRs against their parent Issue and Epic contracts on GitHub (gh) and GitLab (glab).
  Publishes self-contained inline diff suggestions and formal review verdicts. Triggers:
  "forge-review #154", "review PR #42", "audit MR !12", "review branch against main", "forge-review".
---

# Forge-Review — PR/MR Deep Audit & Review Engine

## Scheduling

### Goal
Perform an exhaustive, multi-pass alignment, quality, and security audit of a PR/MR (or branch) against its closing Issue and parent Epic contracts. Present structured evidence in chat, save an immutable audit artifact on disk, and interactively submit formal reviews with self-contained, line-level inline diff suggestions to GitHub (`gh`) or GitLab (`glab`).

### Intent signature
- User invokes `/forge-review` or asks to review a PR, MR, or issue PR.
- User says `forge-review #154`, `review PR #42`, `audit MR !10`, or `review this branch`.
- User wants to verify that a PR satisfies every acceptance criterion in an `epic-forge` issue.
- User wants an automated, deep security and quality audit published directly to the forge diff.

### When to use
- Reviewing an `issue-autopilot` or `gardener` PR on GitHub or GitLab.
- Validating PR alignment against `epic-forge` issues and parent epic architecture.
- Performing a comprehensive 9-dimension diff audit with security checks before merge.
- Publishing batch inline diff comments and formal review verdicts to the forge.

### When NOT to use
- Reviewing purely local, uncommitted files -> use `deep-review` or `review`.
- Writing or implementing the code fixes -> use `ultrawork` or `work`.
- Creating the issues and epics -> use `epic-forge`.
- Automated batch merge without detailed review -> use `gardener-harvest`.

### Expected inputs
- `target`: Issue number (`#154`), PR number (`#42` / `!42`), PR URL, branch name, or none (current branch).
- `mode`: Auto-detected (`issue-pr` = closes issue; `standalone-pr` = no issue; `branch` = local branch vs main).
- `forge`: GitHub (`gh`) or GitLab (`glab`), auto-detected via `git remote get-url origin`.

### Expected outputs
- Raw audit findings at `.agents/results/raw-review-{n}-{sessionId}.md`.
- Pristine, verified audit artifact at `.agents/results/review-pr-{n}-{sessionId}.md`.
- Staged unified diff saved at `.agents/results/diff-pr-{n}.patch` (or `.agents/results/diff-pr-{n}-{sessionId}.patch`).
- Issue specification saved at `.agents/results/spec-issue-{n}.md` (or `.agents/results/spec-issue-{n}-{sessionId}.md`).
- Alignment scorecard and 9-dimension deep review summary in chat.
- User-approved formal Forge Review (`REQUEST_CHANGES`, `APPROVE`, `COMMENT`).
- Line-level inline diff comments with self-contained ` ```suggestion ` blocks.

### State file format (`.agents/results/forge-review/<sessionId>/state.json`)
```json
{
  "session_id": "<sessionId>",
  "forge": "github|gitlab",
  "repo": "owner/repo",
  "target_type": "pr|issue|branch",
  "pr_number": 42,
  "issue_number": 154,
  "epic_number": 12,
  "diff_file": ".agents/results/diff-pr-42.patch",
  "spec_file": ".agents/results/spec-issue-154.md",
  "subagents": {
    "qa_alignment": {
      "status": "pending|running|complete|failed",
      "result_file": ".agents/results/result-qa-alignment-pr-42-<sessionId>.md"
    },
    "deep_review": {
      "status": "pending|running|complete|failed",
      "result_file": ".agents/results/result-deep-review-pr-42-<sessionId>.md"
    },
    "security_audit": {
      "status": "pending|running|complete|failed",
      "result_file": ".agents/results/result-security-audit-pr-42-<sessionId>.md"
    },
    "review_verifier": {
      "status": "pending|running|complete|failed",
      "raw_input_file": ".agents/results/raw-review-42-<sessionId>.md",
      "result_file": ".agents/results/review-pr-42-<sessionId>.md"
    }
  },
  "raw_review_file": ".agents/results/raw-review-42-<sessionId>.md",
  "review_summary_file": ".agents/results/review-pr-42-<sessionId>.md",
  "verdict": "REQUEST_CHANGES|APPROVE|COMMENT",
  "comments_staged": 5,
  "comments_published": 5,
  "published_review_id": "12345678"
}
```

### Dependencies
- Forge CLI (`gh` or `glab`) authenticated on `PATH`.
- `_shared/runtime/providers.md` — shared command map for `gh` / `glab`.
- `_shared/runtime/execution-protocol.md` — File-First State I/O & chat return contract.
- `_shared/core/clarification-protocol.md` — clarification rules & human approval gating.
- `_shared/core/context-loading.md` — dynamic resource loading.
- Subagent skills: `skills/deep-review/SKILL.md`, `skills/review/SKILL.md`, `skills/deepsec/SKILL.md`.
- Local resources: `resources/comment-template.md`, `resources/review-template.md`, `resources/execution-protocol.md`.

### Control-flow features
- Branching by forge provider (`gh` vs `glab`).
- Branching by review target (Issue-linked PR vs Standalone PR vs Local Branch).
- Two-stage 4-Agent Architecture: 3 parallel detector subagents (Contract Alignment, 9-Dimension Deep QA, Security Audit) followed by 1 independent codebase verification & hunk-checking subagent (`review-verifier`).
- Hard Human Approval Gate (`ask_question` / user prompt) before any forge mutation or review publication.
- Idempotency guard and rate-limiting backoff for batch inline review comments.

---

## Structural Flow

### Entry
1. **Detect Forge Provider**: Run `git remote get-url origin` per `providers.md`.
   - Contains `github.com` -> provider=`github`, CLI=`gh`.
   - Contains `gitlab.com` or self-hosted GitLab -> provider=`gitlab`, CLI=`glab`.
   - Neither -> ask user via `ask_question`.
2. **Verify Authentication**: Execute a functional check (`gh repo view` or `glab repo view`). Abort immediately if the functional check fails with instructions to authenticate.
3. **Resolve Target**:
   - Issue number provided (e.g. `#154`): Locate linked PR via forge CLI query or branch convention.
   - PR number provided (e.g. `#42` or `!42`): Resolve PR details, head/base branches, and extract closing issue references (`Closes #X`, `Fixes #X`).
   - Branch provided (e.g. `feat/login`): Compare against base branch (`main` / default).
   - No target provided: Detect current branch; if on default branch, prompt user for PR or Issue number.

### Scenes

1. **ACQUIRE**:
   - Fetch the full unified diff of the PR/branch and write to `.agents/results/diff-pr-{n}.patch` (or `.agents/results/diff-pr-{n}-{sessionId}.patch`).
   - If linked to an issue, fetch the full issue specification, acceptance criteria, and parent epic details; write to `.agents/results/spec-issue-{n}.md` (or `.agents/results/spec-issue-{n}-{sessionId}.md`).
   - Initialize the run state file at `.agents/results/forge-review/<sessionId>/state.json`.

2. **DELEGATE_AUDIT**:
   - Dispatch 3 specialized detector subagents concurrently using `invoke_subagent` (or `task` tool) following the Subagent Delegation Contract (anti-context-dilution):
     - **Subagent 1 (Contract Alignment Agent / `qa-agent`)**:
       - Loads `skills/review/SKILL.md`.
       - Reads `DIFF_FILE` and `SPEC_FILE` from disk.
       - Validates every acceptance criterion, DTO schema, and contract requirement against `file:line` citations in the diff.
       - Saves audit report to `.agents/results/result-qa-alignment-pr-{n}-{sessionId}.md`.
     - **Subagent 2 (9-Dimension Deep QA Reviewer / `deep-reviewer`)**:
       - Loads `skills/deep-review/SKILL.md`.
       - Audits the complete unified diff across 9 dimensions: (1) Logic & Correctness, (2) Regressions & Breaking Changes, (3) State & Concurrency, (4) UI & Accessibility, (5) Test Coverage & Quality, (6) Dead Code & Hygiene, (7) Performance & N+1 Queries, (8) DRY & Architecture Alignment, (9) Checklist & Contract Validation.
       - Stages inline line-level suggestions formatted using `resources/comment-template.md`.
       - Saves findings and staged inline comments to `.agents/results/result-deep-review-pr-{n}-{sessionId}.md`.
     - **Subagent 3 (Security & Authorization Auditor / `security-agent`)**:
       - Loads `skills/review/SKILL.md` and/or `skills/deepsec/SKILL.md`.
       - Audits OWASP Top 10 vulnerabilities (SQLi, XSS, SSRF, IDOR), authorization decorators/middleware, secret leakage, and input sanitization.
       - Saves security report and staged security suggestions to `.agents/results/result-security-audit-pr-{n}-{sessionId}.md`.

3. **RAW_SYNTHESIZE**:
   - Orchestrator waits reactively for all 3 detector subagents to complete.
   - Reads the 3 result files from disk.
   - Writes combined raw findings and unvetted suggestions to `.agents/results/raw-review-{n}-{sessionId}.md` without dropping or preemptively editing findings.

4. **DELEGATE_VERIFICATION**:
   - Orchestrator dispatches **Subagent 4 (`review-verifier`)** via `invoke_subagent`.
   - Subagent 4 independently loads `.agents/results/raw-review-{n}-{sessionId}.md`, the diff patch, and directly inspects the target codebase repository files:
     - Verifies and grounds every finding against actual source code and PR diff hunks.
     - Checks diff hunks: demotes findings on untouched or unmodified lines to top-level summary findings (preventing invalid inline comments outside diff hunks).
     - Validates and fixes ` ```suggestion ` replacement block syntax, ensuring exact whitespace matching and syntactically sound replacements.
     - Deduplicates overlapping findings across the 3 detector subagents.
     - Drops false positives with documented justification.
     - Writes pristine, verified review deliverable at `.agents/results/review-pr-{n}-{sessionId}.md` conforming to `resources/review-template.md`.
     - Determines the verified overall review verdict:
       - 🔴 `REQUEST_CHANGES`: Any verified Critical/High security issue, broken acceptance criteria, or severe functional regression.
       - 🟡 `COMMENT`: Medium/Low non-blocking improvements or questions.
       - 🟢 `APPROVE`: All acceptance criteria met with high quality, no Critical/High findings, passing tests.

5. **PRESENT & GATE (Human Approval Gate)**:
   - Orchestrator loads pristine `.agents/results/review-pr-{n}-{sessionId}.md`.
   - Displays verified Alignment Table, Scorecard, and Staged Suggestions directly in chat.
   - Executes the single Human Approval Gate via `ask_question`:
     - Option 1 (Recommended): `Submit formal review ({VERDICT}) and publish {N} inline diff suggestions to the forge.`
     - Option 2: `Submit top-level summary review only (do not publish inline comments).`
     - Option 3: `Edit/revise staged review before publishing.`
     - Option 4: `Abort without publishing to the remote forge.`
   - **STRICT FORBIDDEN ACTION**: The orchestrator MUST NOT publish reviews or post comments to the remote forge without explicit user confirmation.

6. **PUBLISH**:
   - Upon receiving user approval, execute forge API operations per `_shared/runtime/providers.md`:
     - **GitHub (`gh`)**:
       - Submit batch review with top-level body and line comments via `gh api /repos/{owner}/{repo}/pulls/{n}/reviews`:
         ```json
         {
           "commit_id": "<latest_commit_sha>",
           "body": "<review_summary_markdown>",
           "event": "REQUEST_CHANGES|APPROVE|COMMENT",
           "comments": [
             {
               "path": "src/api/auth.ts",
               "line": 42,
               "side": "RIGHT",
               "body": "### 🔴 CRITICAL: Missing Authorization Guard\n...\n```suggestion\n@UseGuards(AuthGuard)\n```"
             }
           ]
         }
         ```
       - If batch review payload exceeds forge size limits, submit top-level review first, followed by individual discussion comments with rate-limit pacing.
     - **GitLab (`glab`)**:
       - Submit top-level review note: `glab mr note {n} --message "<review_summary_markdown>"`.
       - Create discussion threads for inline suggestions via `glab api projects/:pid/merge_requests/:iid/discussions` with position metadata (`base_sha`, `head_sha`, `start_sha`, `new_path`, `new_line`).
   - Update `.agents/results/forge-review/<sessionId>/state.json` with published review IDs and timestamps.

7. **FINALIZE**:
   - Verify that the review and comments are live on the forge.
   - Provide clickable links to the published PR review and discussion threads.
   - Report immutable artifact paths on disk.
   - Output standard 4-line chat completion summary.

---

### Multi-Agent Topology

```
                              ┌──────────────────────────────────┐
                              │        MAIN ORCHESTRATOR         │
                              │  (`forge-review` Coordinator)    │
                              └─────────────────┬────────────────┘
                                                │
                 ┌──────────────────────────────┼──────────────────────────────┐
                 │ (Stage 1: Dispatched in Parallel)                           │
                 ▼                              ▼                              ▼
     ┌───────────────────────┐      ┌───────────────────────┐      ┌───────────────────────┐
     │   SUBAGENT 1 (Task)   │      │   SUBAGENT 2 (Task)   │      │   SUBAGENT 3 (Task)   │
     │   Contract Alignment  │      │  9-Dimension Deep QA  │      │   Security Auditor    │
     │   (Loads `review`)    │      │ (Loads `deep-review`) │      │   (Loads `deepsec`)   │
     └───────────┬───────────┘      └───────────┬───────────┘      └───────────┬───────────┘
                 │                              │                              │
                 └──────────────────────────────┼──────────────────────────────┘
                                                ▼
                              ┌──────────────────────────────────┐
                              │      Scene 3: RAW_SYNTHESIZE     │
                              │ `.agents/results/raw-review.md`  │
                              └─────────────────┬────────────────┘
                                                │
                                                ▼ (Stage 2: Verification Dispatch)
                              ┌──────────────────────────────────┐
                              │   SUBAGENT 4 (`review-verifier`) │
                              │   Codebase Grounding, Hunk Check,│
                              │   Syntax Fixes, & Deduplication  │
                              └─────────────────┬────────────────┘
                                                │
                                                ▼
                              ┌──────────────────────────────────┐
                              │    PRISTINE REVIEW ARTIFACT      │
                              │ `.agents/results/review-pr.md`   │
                              └─────────────────┬────────────────┘
                                                │
                                                ▼
                              ┌──────────────────────────────────┐
                              │   Scene 5: PRESENT & GATE        │
                              │   (Human Approval Gate)          │
                              └─────────────────┬────────────────┘
                                                │
                                                ▼
                              ┌──────────────────────────────────┐
                              │   Scene 6: PUBLISH & FINALIZE    │
                              │   (Forge API Submission)         │
                              └──────────────────────────────────┘
```

### Transitions
- If target is a branch without an open PR -> run local diff audit; save review artifact; prompt user whether to open a PR or output chat report only.
- If linked issue cannot be found -> switch to `standalone-pr` mode (audit against PR description and general code quality standards).
- If forge API rate limit (429) occurs -> back off exponentially, pause, and resume from `state.json`.
- If user rejects review publication -> keep artifact saved on disk, do not call Forge mutation APIs, exit cleanly.

### Failure and Recovery
| Failure Mode | Root Cause | Recovery Procedure |
|:---|:---|:---|
| Auth failure | Expired/missing forge token | Prompt user to run `gh auth login` or `glab auth login`; abort before analysis. |
| Linked PR not found | Branch naming or issue links mismatch | Ask user for explicit PR number or branch name via `ask_question`. |
| Diff exceeds context limit | Massive PR (>2000 lines diff) | Subagents chunk diff by file/module boundaries; inspect high-risk files first. |
| Forge rejects inline line position | File renamed or line offset shifted | Verify commit SHA match; fallback to posting comment at file-level or top-level review. |
| Secondary rate limit / 429 | Too many rapid API requests | Exponential backoff (5s, 15s, 45s); pace inline comment submissions; resume from `state.json`. |
| Subagent audit timeout or failure | Process error in subagent | Resend prompt or re-spawn single failed subagent; do not re-run passed subagents. |
| Verification hunk mismatch | Detector hallucinated line or target outside hunk | Subagent 4 demotes finding to top-level review body, preserving audit trail without API failure. |

### Exit
- **Success**: PR/MR diff and contract thoroughly audited across all 3 detector domains, verified and grounded by Subagent 4; structured report synthesized; user approval obtained; formal review and inline suggestions published to remote forge; URLs reported.
- **Partial Success**: Full audit and verification completed and artifact saved locally, but forge publication skipped or declined by user.
- **Failure**: Unrecoverable authentication error, missing target PR/diff, or fatal API failure; failure state recorded in `state.json`.

---

## Logical Operations

### Actions
| Action | SSL Primitive | Evidence |
|:---|:---|:---|
| Detect provider & auth | `READ` | `_shared/runtime/providers.md`, `git remote get-url origin`, `gh/glab repo view` |
| Resolve review target | `INFER` / `REQUEST` | PR metadata, closing keywords (`Closes #N`), `ask_question` |
| Fetch diff & issue spec (Scene 1) | `READ` / `WRITE` | `gh pr diff`, `gh issue view`, `glab mr diff`, `.agents/results/diff-pr-*.patch` |
| Dispatch detector subagents (Scene 2) | `TRANSFER` | `invoke_subagent` for Subagents 1, 2, 3 in parallel |
| Audit contract alignment | `COMPARE` / `VALIDATE` | Subagent 1 report: `.agents/results/result-qa-alignment-pr-*.md` |
| 9-dimension deep review | `VALIDATE` | Subagent 2 report: `.agents/results/result-deep-review-pr-*.md` |
| Security & OWASP audit | `VALIDATE` | Subagent 3 report: `.agents/results/result-security-audit-pr-*.md` |
| Synthesize raw review (Scene 3) | `WRITE` | `.agents/results/raw-review-{n}-{sessionId}.md` |
| Dispatch verification subagent (Scene 4) | `TRANSFER` | `invoke_subagent` for Subagent 4 (`review-verifier`) |
| Codebase grounding & hunk verification | `VALIDATE` / `COMPARE` | Subagent 4 diff hunk check, syntax validation, deduplication |
| Write pristine review artifact | `WRITE` | `.agents/results/review-pr-{n}-{sessionId}.md` |
| Human approval gate (Scene 5) | `VALIDATE` / `REQUEST` | Chat presentation & `ask_question` interactive decision |
| Publish formal review (Scene 6) | `CALL_TOOL` | Forge CLI / API mutation (`gh api`, `glab api`) |
| Finalize and report completion (Scene 7) | `NOTIFY` | Chat output with URLs and standard 4-line completion summary |

### Tools and Instruments
- `gh` (GitHub CLI) & `glab` (GitLab CLI) per `_shared/runtime/providers.md`.
- `invoke_subagent` / `task` tool for parallel and verification subagent execution.
- `ask_question` tool for interactive human approval gates.
- `view_file` / `write_to_file` / `replace_file_content` for file-first state management.
- `resources/comment-template.md` for inline diff suggestions.
- `resources/review-template.md` for top-level review synthesis.

### Canonical Workflow Path
```bash
# 1. Detect provider & verify auth
git remote get-url origin
gh repo view || glab repo view

# 2. Acquire diff and issue contract to disk (Scene 1: ACQUIRE)
gh pr diff 42 > .agents/results/diff-pr-42.patch
gh issue view 154 --json title,body,labels > .agents/results/spec-issue-154.md

# 3. Delegate audits to 3 detector subagents in parallel (Scene 2: DELEGATE_AUDIT)
# Subagents write results to .agents/results/result-*-pr-42-${SESSION_ID}.md

# 4. Synthesize raw review findings (Scene 3: RAW_SYNTHESIZE)
# Orchestrator compiles raw findings into .agents/results/raw-review-42-${SESSION_ID}.md

# 5. Delegate verification to Subagent 4 (Scene 4: DELEGATE_VERIFICATION)
# Subagent 4 (review-verifier) inspects codebase, checks diff hunks, fixes syntax, and writes:
# .agents/results/review-pr-42-${SESSION_ID}.md

# 6. Present in chat & execute Human Approval Gate via ask_question (Scene 5: PRESENT & GATE)

# 7. Publish approved review and inline suggestions to Forge API (Scene 6: PUBLISH)
gh api /repos/{owner}/{repo}/pulls/42/reviews --input payload.json

# 8. Finalize and return 4-line chat summary (Scene 7: FINALIZE)
```

### Resource Scope
| Scope | Resource Target |
|:---|:---|
| `CODEBASE` | Target repository git remote, PR branches, and working tree (read-only audit; no source edits) |
| `LOCAL_FS` | `.agents/results/` (diff patches, issue specs, subagent audit results, raw review, review summary, `state.json`) |
| `PROCESS` | `gh` / `glab` CLI processes and git commands |
| `NETWORK/FORGE` | GitHub REST/GraphQL API or GitLab API endpoints |

### Preconditions
- Forge CLI (`gh` or `glab`) installed and authenticated on `PATH`.
- Read access to target repository, PR diff, and associated issue/epic.
- Write access to target repository for submitting PR reviews and inline comments.
- Working directory is inside a valid git repository with remote `origin` configured.

### Effects and Side Effects
- Audit artifacts, diff patches, raw findings, and verified review written to `.agents/results/`.
- State file updated at `.agents/results/forge-review/<sessionId>/state.json`.
- Formal review status and comments posted to remote forge (only after explicit human approval).
- Zero source code files modified in the repository.

### Guardrails
1. **Human Approval Gate (NON-NEGOTIABLE)**: Never publish reviews, approve PRs, request changes, or post inline comments to the remote forge without presenting the staged review and receiving explicit confirmation via `ask_question`.
2. **Anti-Context-Dilution Subagent Delegation**: Orchestrator is strictly forbidden from performing unified diff audits directly in the main context. Audits must be delegated to specialized subagents.
3. **Mandatory 100% Subagent 4 Verification**: 100% of raw findings from Subagents 1, 2, and 3 MUST be independently verified, hunk-checked, syntax-validated, and grounded against real codebase files by Subagent 4 (`review-verifier`) before review presentation. Raw, unverified findings must never be presented directly to the user or published to the forge.
4. **File-First State I/O**: Pass artifacts between orchestrator and subagents via disk references (`.agents/results/`), never via bloated in-memory prompt strings.
5. **Self-Contained Inline Suggestions & Hunk Alignment**: All inline comments proposing code changes MUST include syntactically valid, self-contained ` ```suggestion ` replacement blocks that fall strictly within modified diff hunks. Untouched lines must be demoted to top-level summary comments.
6. **Rigorous Evidence Citation**: Every acceptance criterion and defect finding must cite explicit `file:line` references from the diff.

---

## References
- Local comment template: `.agents/skills/forge-review/resources/comment-template.md`
- Local review template: `.agents/skills/forge-review/resources/review-template.md`
- Local execution protocol: `.agents/skills/forge-review/resources/execution-protocol.md`
- Shared forge provider CLI mappings: `.agents/skills/_shared/runtime/providers.md`
- Shared execution protocol: `.agents/skills/_shared/runtime/execution-protocol.md`
- Shared clarification protocol: `.agents/skills/_shared/core/clarification-protocol.md`
- Shared context loading: `.agents/skills/_shared/core/context-loading.md`
- Subagent skill (Deep Review): `.agents/skills/deep-review/SKILL.md`
- Subagent skill (QA Review): `.agents/skills/review/SKILL.md`
- Subagent skill (Security Audit): `.agents/skills/deepsec/SKILL.md`
- Parent skill (Epic Decomposition): `.agents/skills/epic-forge/SKILL.md`
