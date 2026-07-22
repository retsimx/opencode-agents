# Subagent Assessment Prompt

Use this prompt verbatim when spawning a `task` subagent to assess a single PR.

**Inputs to substitute before spawning:**
- `<number>` — the PR number
- `<repo_path>` — absolute path to the local repo
- `<provider>` — `github` or `gitlab`
- `<high_risk_globs>` — newline-joined list from `.agents/skills/gardener-harvest/resources/repo-rules.yaml`

The subagent reports raw findings ONLY. The parent agent applies the decision
logic. The subagent does not recommend MERGE/ASK/SKIP, does not mutate the
repo, and does not run any write command.

---

Assess PR #<number> in the repository at `<repo_path>`.

You are read-only. Do NOT push, rebase, retry CI, edit files, or merge. Stick
to fetch / read commands.

## Step 1: Gather evidence

Provider: `<provider>`

GitHub (`gh`) commands:

```bash
gh pr diff <number>
gh pr checks <number>
gh pr view <number> --json additions,deletions,changedFiles,headRefName,state,isDraft,title
```

GitLab (`glab`) commands (adapt project id `:pid` and merge request iid `:iid` from the local repo):

```bash
glab mr diff <number>
glab api projects/:pid/merge_requests/:iid/pipelines
glab mr view <number> --output json
```

## Step 2: Analyze

### Risk classification

Classify the PR into exactly one category:

**LOW**
- Mechanical cleanup (unused imports, dead code, formatting)
- Comment or documentation only
- Trivial refactor with no behavioural change (rename variable, extract helper)

**MEDIUM**
- Logic modification (if-statement restructuring, loop changes)
- Query changes (SQL, ORM querysets)
- State machine or validation changes
- CI/workflow changes (only when not already flagged as a repo-specific high risk)

**HIGH**
- Authentication or permissions code
- Security-sensitive changes
- Database migrations
- Infrastructure changes
- API serializer or endpoint behaviour changes
- Concurrent or distributed logic

Rule: if the diff touches any file matching the high-risk globs below, escalate one level above the natural classification (LOW → MEDIUM → HIGH).

High-risk globs for this repo:
```
<high_risk_globs>
```

### Behaviour change

Answer exactly one:

- **YES** — runtime behaviour changes (output, side effect, state transition differs)
- **NO** — purely cosmetic or structural, no runtime difference
- **UNCLEAR** — hard to determine from diff alone

Rule: If YES, classification cannot be LOW.

### CI status

Report counts exactly:

```
Checks Passed: X
Checks Failed: Y
Checks Pending: Z
```

If any check failed (Y > 0), list the failing check names below the counts.

Note: optional checks count. Do NOT silently treat "optional check failed but required checks green" as a clean run — report the counts; the parent decides.

### Change size

```
Files changed: N
Lines added: +X
Lines removed: -Y
```

### Scope coherence

Can every changed line be justified by the PR title?

- **YES** — every changed line aligns with the title
- **NO** — at least one change seems unrelated (scope creep)

If NO, list the unrelated changes with file:line references.

### Repository-specific flags

Load the high-risk globs list above. For every changed file, check each glob.
Report every match as a flag, with the affected file path.

If no flags: report `None`.

### Confidence

Assign a confidence percentage (0–100%) based on:

- Clarity of the diff (small, mechanical = higher)
- CI completeness (all pass with no pending = higher)
- Similar patterns in git history (more matches = higher)
- Absence of edge cases

Examples:
- Obvious import cleanup, CI green, many similar merged PRs → 95%
- Cosmetic refactor with adequate test coverage → 80%
- Logic change with thin test coverage → 60%
- Complex state change, unclear tests → 40%

To check git history for similar merged patterns, grep the merged-PR commit
log for short tokens extracted from the PR title (e.g. "import", "rename",
"extract helper", "SIM105", "ARG002"). Example:

```bash
git log --oneline --merges --grep="<token>" --max-count=20
# or
git log --oneline --all --grep="<token>" --max-count=20
```

Substitute `<token>` with up to three short, distinctive words from the PR title. Do NOT grep for the literal string "similar pattern".

### Large diff

If `gh pr diff <number>` (or glab equivalent) output exceeds ~50,000 characters,
record confidence ≤ 30 and report `large_diff: true`. Do not attempt to read
or summarize the rest of the diff.

## Step 3: Report

Use this exact format. No decision logic, no merge recommendation — those
are the parent agent's job.

```
PR #<number>

Classification: LOW/MEDIUM/HIGH

Behavior change:
YES/NO/UNCLEAR

CI:
Checks Passed: X
Checks Failed: Y
Checks Pending: Z
[List failing checks if any]

Files changed: N
Lines added: +X
Lines removed: -Y

Scope coherence:
YES/NO
[If NO, list unrelated changes]

Repository-specific flags:
- [flag 1]
- [flag 2]
- [or "None"]

Confidence: N%

Large diff: true/false

Top concerns:
- [concern 1]
- [concern 2]
- [or "None"]
```

Stop after the report. Do not run any further commands. Do not attempt to
merge, rebase, push, or modify the PR.