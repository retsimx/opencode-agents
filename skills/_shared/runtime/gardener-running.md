# Gardener Family — How to Run

Pipeline: **`gardener-sow` → `gardener-tend` → `gardener-harvest`**.

Each skill is **single-shot**. Wrap invocations in an outer shell loop so every
run starts with a fresh context. Do not loop inside the skill.

**Only one gardener skill at a time** against the same `CONTROL_ROOT` +
`MAIN_REPO`. Do not run sow and tend in parallel.

## Prerequisites

- Target repo as cwd (`MAIN_REPO`)
- Forge CLI authenticated (`gh` or `glab`)
- Agent pack containing `skills/gardener-sow` (`CONTROL_ROOT`)

```bash
cd /path/to/repo
export MODEL="${MODEL:-qwen-adacs/qwen}"
```

## gardener-sow

One draft micro-improvement, then exit.

```bash
while :; do
  timeout 3600 opencode run --model "$MODEL" \
    "/gardener-sow - execute exactly one iteration then end the turn and stop."
done
```

## gardener-tend

One maintenance action on the oldest gardener PR that needs work, then exit.

```bash
while :; do
  timeout 3600 opencode run --model "$MODEL" "/gardener-tend"
done
```

## gardener-harvest

One oldest non-draft gardener PR: merge, comment, close, or skip, then exit.

```bash
while :; do
  timeout 3600 opencode run --model "$MODEL" "/gardener-harvest"
  sleep 60
done
```

## Suggested scheduling

Run **one** of these loops at a time (separate machines/repos are fine).

| Cadence | Skill |
|---------|-------|
| Continuous | `gardener-sow` **or** `gardener-tend` (not both) |
| On demand / slower poll | `gardener-harvest` alone |

## Runtime artifacts

| Artifact | Purpose |
|----------|---------|
| `$CONTROL_ROOT/state/gardener/open.json` | Open PR cache |
| `$CONTROL_ROOT/state/gardener/closed.json` | Human-rejection learning |
| `$CONTROL_ROOT/state/gardener/intent.json` | Crash recovery |
| `$CONTROL_ROOT/worktrees/gardener-<run-id>` | Isolated edit checkout |
| Branch `gardener/run-<run-id>` | Per-shot branch |
| Title `chore(gardener): …` | Identification |

## Related

- `.agents/skills/_shared/runtime/gardener-contract.md`
- `.agents/skills/_shared/runtime/gardener-state.md`
- `.agents/skills/_shared/runtime/providers.md`
