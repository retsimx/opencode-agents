# Gardener State

Files under `$CONTROL_ROOT/state/gardener/`.

## Write rule

Validate the complete JSON, write a sibling temp file, atomically replace.
Never overwrite invalid `closed.json`. If `open.json` is absent, invalid, or
for a different repository, rebuild it from this repo's gardener PR metadata
(do not keep the old items). If `closed.json` is absent, start `{ "schema_version": 1, "repository": "<id>", "provider": "<p>", "items": [] }`. If `closed.json` is present but invalid, stop.

`repository` is `provider-host/owner/project` from `origin`.

## intent.json (crash recovery, not a lock)

```json
{
  "skill": "sow|tend|harvest",
  "repository": "provider-host/owner/project",
  "operation": "ship_draft_pr",
  "subject": "PR number or branch",
  "expected_result": "what must be true on the forge or git",
  "worktree_path": "",
  "run_dir": "",
  "run_id": "",
  "pr_number": null,
  "head_sha": "",
  "base_sha": ""
}
```

Operations: `ship_draft_pr`, `tend_rebase`, `tend_ci`, `tend_comment`,
`tend_close`, `tend_promote`, `harvest_merge`, `harvest_needs_fix`,
`harvest_close`.

Write the intent immediately before the external effect. Query actual git/forge
state before retrying. Delete the intent only after the whole expected result
is verified.

On entry: if `intent.json` exists and `skill` is this skill, reconcile it. If
`skill` is a different gardener skill, stop and report that skill (another
invocation is mid-recovery).

## open.json

Recoverable cache. Rebuild from a full paginated list of open gardener PRs
(metadata only). Do not fetch every body. Fields per item:

`number`, `url`, `title`, `branch`, `head_sha`, `topic`, `rationale`, `areas`,
`created_at`.

`topic`, `rationale`, and `areas` are filled when sow/tend/harvest has that PR's
body. Do not invent them from the title alone.

## closed.json

Durable learning. Categories:

- `rejected` — a **non-marked** comment explicitly asked to reject or close.
  `suppress_equivalent` is true.
- `unknown` — closed without that human ask (includes agent-only CLOSE).
  `suppress_equivalent` is false.

Do not use other categories.

## Suppression

The scanner cites only rows it believes collide (row `number` + one sentence).
The parent walks open items and closed `rejected` items (title and `areas`
first; hydrate a cited row's body if needed). Suppress if **either**:

- the scanner cited a colliding open or `rejected` row and the parent agrees, or
- the parent finds a colliding open or `rejected` row the scanner missed.

No guesswork phrases. "Resembles" is not a rule. Reproposal against `unknown`
is allowed if the new PR names the prior PR and explains why.

## Worktree cleanup (entry)

`git worktree list`, then remove any path under `$CONTROL_ROOT/worktrees/`
matching `gardener-*` that is **not** `intent.worktree_path`.

## Harvest temp cleanup (entry)

Delete `$CONTROL_ROOT/results/tmp/pr-harvest/*` directories that are **not**
`intent.run_dir`.
