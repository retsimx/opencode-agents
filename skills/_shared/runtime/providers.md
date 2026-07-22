# Provider Command Mappings (GitHub / GitLab)

Canonical CLI map for skills that talk to a forge. Use **PR** as the generic
term for GitHub pull requests and GitLab merge requests.

Skills that depend on this file: `issue-autopilot`, `gardener-sow`, `gardener-tend`,
`gardener-harvest`.

## Detect provider

```bash
git remote get-url origin
```

- URL contains `github.com` → provider=`github`, CLI=`gh`
- URL contains `gitlab.com` **or** a self-hosted GitLab hostname → provider=`gitlab`, CLI=`glab`
- Neither matches → abort and ask the user which provider + CLI to use

Resolve once at skill Entry. Pass `PROVIDER` (`github`|`gitlab`) to every
subagent. Never hardcode `gh` or `glab` outside this table.

## Auth check

| Provider | Command |
|----------|---------|
| GitHub | `gh auth status` |
| GitLab | `glab auth status` |

Abort immediately on failure. Do not attempt to log in for the user.

- GitHub: `gh auth login` with `repo` scope (and `workflow` if inspecting workflow runs).
- GitLab: `glab auth login` with `api` scope on the target instance.

## Issues

| Operation | GitHub (`gh`) | GitLab (`glab`) |
|-----------|---------------|-----------------|
| View issue | `gh issue view N --json title,body,labels,comments,assignees,state` | `glab issue view N --output json` (normalize title/body/labels/assignees/state client-side; comments via notes API if missing) |
| Comment on issue | `gh issue comment N --body-file PATH` | `glab issue note N --message "$(cat PATH)"` (or `--message-file` if available) |
| List open issues | `gh issue list --state open --json number,title` | `glab issue list --output json` then filter `state == "opened"` |

Issue close keywords in commit/PR body (`Closes #N`, `Fixes #N`) work on both forges when the default branch receives the merge.

## Pull / merge requests

| Operation | GitHub (`gh`) | GitLab (`glab`) |
|-----------|---------------|-----------------|
| List open PRs (drafts included — filter client-side) | `gh pr list --state open --json number,title,headRefName,baseRefName,createdAt,isDraft,mergeable --limit 1000` | `glab mr list --state opened --output json --per-page 100` then page until exhausted |
| View single PR | `gh pr view N --json number,state,isDraft,title,additions,deletions,changedFiles,headRefName,baseRefName,mergeable,mergeableState,url` | `glab mr view N --output json` |
| View by URL | `gh pr view "<URL>" --json state` | `glab mr view "<URL>" --output json` |
| Fetch diff | `gh pr diff N` | `glab mr diff N` |
| Create draft PR | `gh pr create --draft --base main --title "..." --body-file PATH` | `glab mr create --draft --target-branch main --title "..." --description "$(cat PATH)"` (or `--description-file` if available) |
| Promote draft → ready | `gh pr ready N` | `glab mr update N --ready` |
| Fetch CI / pipeline status | `gh pr checks N` | `glab api projects/:pid/merge_requests/:iid/pipelines` — use latest pipeline `status` (`success`/`failed`/`running`/`pending`/`canceled`) |
| Merge (squash + delete branch) | `gh pr merge N --squash --delete-branch` | `glab mr merge N --squash --remove-source-branch` |
| Verify merge state | `gh pr view N --json state` (expect `MERGED`) | `glab mr view N --output json` (expect `state == "merged"`) |
| Verify branch deleted | `gh api repos/{owner}/{repo}/branches/{branch}` (expect 404) | `glab api projects/:pid/repository/branches/{branch}` (expect 404) |

### Normalize list/view fields (client-side)

Always map provider JSON into this shape before skill logic:

```
number, title, headRefName, baseRefName, createdAt, isDraft, url, state, mergeableState
```

| Normalized | GitHub | GitLab |
|------------|--------|--------|
| `number` | `number` | `iid` |
| `headRefName` | `headRefName` | `source_branch` |
| `baseRefName` | `baseRefName` | `target_branch` |
| `createdAt` | `createdAt` | `created_at` |
| `isDraft` | `isDraft` | `draft` OR `work_in_progress` |
| `url` | `url` | `web_url` |
| `state` (open) | `OPEN` | `opened` |
| `state` (merged) | `MERGED` | `merged` |
| `state` (closed) | `CLOSED` | `closed` |

### Mergeability (`mergeableState`)

| Normalized action | GitHub `mergeableState` | GitLab `detailed_merge_status` / `merge_status` |
|-------------------|-------------------------|--------------------------------------------------|
| Rebase (conflicts) | `dirty` | `cannot_be_merged`, `conflict` |
| Rebase (behind) | `behind` | treat as behind when HEAD is not up to date with target; if only `can_be_merged` / `mergeable` → clean |
| CI running — skip | `unstable` | `checking`, `unchecked`, pipeline `running`/`pending` |
| Clean — next check | `clean` | `can_be_merged` / `mergeable` |

If GitLab omits a behind signal, compare `git rev-list --count origin/<base>..<head>` locally after fetch; `> 0` means behind.

## Reviewer comments / notes

| Operation | GitHub (`gh`) | GitLab (`glab`) |
|-----------|---------------|-----------------|
| Inline review comments | `gh api /repos/{owner}/{repo}/pulls/N/comments` | `glab api projects/:pid/merge_requests/:iid/discussions` (flatten notes with position) |
| PR conversation comments | `gh api /repos/{owner}/{repo}/issues/N/comments` | `glab api projects/:pid/merge_requests/:iid/notes` |
| Delete a comment/note | `gh api -X DELETE /repos/{owner}/{repo}/issues/comments/ID` or `.../pulls/comments/ID` | `glab api -X DELETE projects/:pid/merge_requests/:iid/notes/:note_id` |
| Reply asking clarification | `gh pr comment N --body "..."` | `glab mr note N --message "..."` |

## GitLab project id (`:pid`)

Resolve once at Entry and cache for the run:

```bash
glab api projects/:encoded-path
# or: glab repo view --output json  → use .id
```

`:iid` is the MR number shown in the UI (same as normalized `number`).

## List / draft / sort rules (both providers)

1. **Always pass an explicit list limit / paginate.** GitHub `gh pr list` defaults to 30 — always `--limit 1000` (or paginate via search API above 1000). GitLab: `--per-page 100 --page N` until empty.
2. **Drafts are included in open lists.** Filter client-side (`isDraft == false` when the skill must skip drafts; keep drafts when the skill maintains them).
3. **Sort oldest-first** by `createdAt` ascending after fetch.
4. **Rate limit:** before large list passes, `gh api rate_limit` or `glab api rate_limit`. On 429, finish in-flight work, save state, exit.

## Local CI config discovery

When a skill must mirror remote CI locally, prefer the file that matches the provider:

| Provider | Primary config | Also check |
|----------|----------------|------------|
| GitHub | `.github/workflows/*.yml` | project docs / `AGENTS.md` |
| GitLab | `.gitlab-ci.yml` | project docs / `AGENTS.md` |

If both exist, use the provider that matched `origin`. Always also follow project-local verify scripts named in `AGENTS.md` / skill resources.

## Merge strategy defaults (when a skill merges)

- Squash + delete source branch only (`--delete-branch` / `--remove-source-branch`).
- If the provider rejects squash, SKIP with `merge_strategy_mismatch` — do not fall back.
- Issue merges **serially** with `sleep 2` between successful merges.

## Terminology for skill prose

- Prefer **PR** (covers GitHub PR and GitLab MR).
- Prefer **forge** or **provider** over "GitHub".
- Prefer **CLI** or name both: `` `gh` / `glab` ``.
- Prefer **issue** (both forges).
- Prefer **draft** (GitLab WIP ≡ draft).
