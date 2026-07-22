# Gardener Family — How to Run

Pipeline: **`gardener-sow` → `gardener-tend` → `gardener-harvest`**.

Each skill is **single-shot** (or one maintenance action / one harvest pass). For continuous work, wrap invocations in an outer shell loop so every run starts with a fresh model context. Do **not** ask the skill to loop forever inside one session.

## Prerequisites

- Repo root as cwd
- Forge CLI authenticated for `origin` (`gh` or `glab` — see `.agents/skills/_shared/runtime/providers.md` in this directory)
- OpenCode (or equivalent) with the project skills loaded
- Optional: set `MODEL` to your preferred OpenCode model id

```bash
cd /path/to/repo
export MODEL="${MODEL:-qwen-adacs/qwen}"
```

## gardener-sow — plant one micro-fix PR

One iteration: scan → work → verify → draft PR → exit.

```bash
clear
while :; do
  timeout 3600 opencode run --model "$MODEL" \
    "/gardener-sow - execute exactly one iteration then end the turn and stop."
done
```

- `timeout 3600` caps a hung shot (adjust as needed).
- Outer `while` is what makes gardening continuous; each shot is independent.
- Ctrl-C stops the outer loop.

One-off (no loop):

```bash
opencode run --model "$MODEL" \
  "/gardener-sow - execute exactly one iteration then end the turn and stop."
```

## gardener-tend — tend one gardener PR

Already one action per invocation (rebase, CI fix, comment, or promote). Outer loop drains the queue.

```bash
clear
while :; do
  timeout 3600 opencode run --model "$MODEL" "/gardener-tend"
done
```

One-off:

```bash
opencode run --model "$MODEL" "/gardener-tend"
```

Filters to open PRs titled `chore(gardener): …`. Exits silently when nothing needs work (outer loop will keep polling).

## gardener-harvest — merge queue pass

Walks open PRs, assesses, merges / asks / skips. Usually run **on demand** (needs a human nearby for MEDIUM asks), not as a tight infinite loop.

```bash
opencode run --model "$MODEL" "/gardener-harvest"
```

Optional slow poll (e.g. after a tend loop has promoted drafts):

```bash
clear
while :; do
  timeout 7200 opencode run --model "$MODEL" "/gardener-harvest"
  sleep 300
done
```

## Suggested scheduling

| Cadence | Skill | Notes |
|---------|-------|-------|
| Continuous / overnight | `gardener-sow` | Outer `while`; produces draft micro-PRs |
| Frequent (every few minutes) | `gardener-tend` | Keeps sow PRs rebase/CI-healthy |
| On demand or sparse | `gardener-harvest` | Human for ASK outcomes |

Example: two terminals — sow in one, tend in another; harvest manually when ready.

## Why outer loops (not in-skill loops)

- Fresh context each shot → less dilution and drift
- `timeout` can kill a stuck agent without killing a long in-process loop’s state machine awkwardly
- Failures are isolated; the next `opencode run` starts clean
- Matches how `gardener-tend` was already designed (one change per invoke)

## Runtime artifacts (unchanged across renames)

| Artifact | Purpose |
|----------|---------|
| `.agents/results/gardener-state.md` | Sow history / open PR tracking |
| Branch `gardener/iter-{NNNN}-{slug}` | Per-shot branch |
| PR title `chore(gardener): …` | Tend filter |
| `.agents/results/pr-merge-queue.json` | Harvest resume state |

## Related

- Provider CLI map: `.agents/skills/_shared/runtime/providers.md` (this directory)
- Skills: `.agents/skills/gardener-sow/SKILL.md`, `.agents/skills/gardener-tend/SKILL.md`, `.agents/skills/gardener-harvest/SKILL.md`
