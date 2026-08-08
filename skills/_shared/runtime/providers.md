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

`gh auth status` / `glab auth status` is **NOT a reliable gate** — both CLIs
sometimes print "Invalid token provided" even when the token works. Verify auth
with a functional request **against the repo** instead:

| Provider | Command |
|----------|---------|
| GitHub | `gh repo view` |
| GitLab | `glab repo view` |

Abort only if the functional request fails. Do not attempt to log in for the user.

- GitHub: `gh auth login` with `repo` scope (and `workflow` if inspecting workflow runs).
- GitLab: `glab auth login` with `api` scope on the target instance.

## CLI invocation rules (both providers)

- **cwd-sensitive**: `gh` and `glab` resolve the target repo from the current
  directory's git remote. Run commands from the repo root (or the worktree) or
  pass `-R OWNER/REPO` / `--repo` explicitly. Running from the wrong directory
  hits the wrong repo or fails (e.g. `gh` from a GitLab repo: "none of the git
  remotes configured for this repository point to a known GitHub host").
- **glab has no `--output json`** (verified on glab 1.36.0): `glab mr list`,
  `glab mr view`, `glab issue list`, `glab issue view`, and `glab repo view` do
  NOT accept `--output json`. For machine-readable output use `glab api` (see
  tables below). `glab mr list` / `glab issue list` default to open items — there
  is no `--state opened` flag.
- **`glab api rate_limit` is not usable** (returns 404; GitLab requires admin for
  `/rate_limit`). On GitLab, handle 429 responses when they occur instead of
  pre-checking. `gh api rate_limit` works on GitHub.

## Issues

| Operation | GitHub (`gh`) | GitLab (`glab`) |
|-----------|---------------|-----------------|
| View issue | `gh issue view N --json title,body,labels,comments,assignees,state` | `glab api projects/:pid/issues/:iid` (normalize title/body/labels/assignees/state client-side; comments via notes API if missing) |
| Comment on issue | `gh issue comment N --body-file PATH` | `glab issue note N --message "$(cat PATH)"` (or `--message-file` if available) |
| List open issues | `gh issue list --state open --json number,title` | `glab api "projects/:pid/issues?state=opened&per_page=100"` then page until exhausted |

Issue close keywords in commit/PR body (`Closes #N`, `Fixes #N`) work on both forges when the default branch receives the merge.

## Pull / merge requests

| Operation | GitHub (`gh`) | GitLab (`glab`) |
|-----------|---------------|-----------------|
| List open PRs (drafts included — filter client-side) | `gh pr list --state open --json number,title,headRefName,baseRefName,createdAt,isDraft,mergeable --limit 1000` | `glab api "projects/:pid/merge_requests?state=opened&per_page=100"` then page until exhausted (`state` may be `opened`/`closed`/`merged`/`all`) |
| View single PR | `gh pr view N --json number,state,isDraft,title,additions,deletions,changedFiles,headRefName,baseRefName,mergeable,mergeStateStatus,url` | `glab api projects/:pid/merge_requests/:iid` |
| View by URL | `gh pr view "<URL>" --json state` | `glab api projects/:pid/merge_requests/:iid` (extract `:iid` from the URL) |
| Fetch diff | `gh pr diff N` | `glab mr diff N` |
| Create draft PR | `gh pr create --draft --base main --title "..." --body-file PATH` | `glab mr create --draft --target-branch main --title "..." --description "$(cat PATH)"` (or `--description-file` if available) |
| Promote draft → ready | `gh pr ready N` | `glab mr update N --ready` |
| Fetch CI / pipeline status | `gh pr checks N` | `glab api projects/:pid/merge_requests/:iid/pipelines` — use latest pipeline `status` (`success`/`failed`/`running`/`pending`/`canceled`) |
| Merge (squash + delete branch) | `gh pr merge N --squash --delete-branch` | `glab mr merge N --squash --remove-source-branch` |
| Verify merge state | `gh pr view N --json state` (expect `MERGED`) | `glab api projects/:pid/merge_requests/:iid` (expect `state == "merged"`) |
| Verify branch deleted | `gh api repos/{owner}/{repo}/branches/{branch}` (expect 404) | `glab api projects/:pid/repository/branches/{branch}` (expect 404) |

### Normalize list/view fields (client-side)

Always map provider JSON into this shape before skill logic:

```
number, title, headRefName, baseRefName, createdAt, isDraft, url, state, mergeStateStatus
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

### Mergeability (`mergeStateStatus` on GitHub)

GitHub field is `mergeStateStatus` (values `CLEAN`/`UNSTABLE`/`DIRTY`/`BEHIND`/`BLOCKED`/`DRAFT`/`UNKNOWN` — verified `CLEAN` and `UNSTABLE` live); `mergeable` is a separate boolean-ish field (`MERGEABLE`/`CONFLICTING`/`UNKNOWN`). GitLab uses `detailed_merge_status` / `merge_status`.

| Normalized action | GitHub `mergeStateStatus` | GitLab `detailed_merge_status` / `merge_status` |
|-------------------|---------------------------|--------------------------------------------------|
| Rebase (conflicts) | `DIRTY` | `cannot_be_merged`, `conflict` |
| Rebase (behind) | `BEHIND` | treat as behind when HEAD is not up to date with target; if only `can_be_merged` / `mergeable` → clean |
| CI running — skip | `UNSTABLE` | `checking`, `unchecked`, pipeline `running`/`pending` |
| Clean — next check | `CLEAN` | `can_be_merged` / `mergeable` |

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
```

`glab repo view` does NOT output JSON — use `glab api projects/:encoded-path` and read `.id`.

`:iid` is the MR number shown in the UI (same as normalized `number`).

## List / draft / sort rules (both providers)

1. **Always pass an explicit list limit / paginate.** GitHub `gh pr list` defaults to 30 — always `--limit 1000` (or paginate via search API above 1000). GitLab: `--per-page 100 --page N` until empty.
2. **Drafts are included in open lists.** Filter client-side (`isDraft == false` when the skill must skip drafts; keep drafts when the skill maintains them).
3. **Sort oldest-first** by `createdAt` ascending after fetch.
4. **Rate limit:** on GitHub, pre-check with `gh api rate_limit`. GitLab has no non-admin rate-limit endpoint (`glab api rate_limit` returns 404) — just handle 429 responses when they occur. On 429, finish in-flight work, save state, exit.

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
