---
name: epic-forge
description: >
  Decompose an approved design/architecture into forge-created epics + issues (dfn-grade
  self-contained contracts), create + link them natively (blocked_by), and update existing
  epics when the design or code evolves. Forge-agnostic (GitHub gh / GitLab glab). Triggers:
  "create epics for <design>", "decompose <architecture> into issues", "update epic #N: <change>",
  "the spec has diverged from the code".
---

# Epic-Forge — Epic & Issue Decomposition to the Forge

## Scheduling

### Goal
Turn an approved design into forge epics + issues with full, self-contained contracts (dfn-grade detail), create + link them natively, and update existing epics when the design or code evolves. Forge-agnostic (GitHub / GitLab).

### Intent signature
- User asks to create epics/issues from a design or architecture.
- User asks to decompose work into dependency-aware issues on the forge.
- User asks to update an existing epic (new feature, small tweak, spec divergence).
- User invokes with no context (skill must ask mode + context first).

### When to use
- A design is approved (or described) and needs to become forge epics + issues.
- An existing epic needs new issues, changed issues, or its spec updated.
- The user wants dfn-grade, self-contained, copy-pasteable issue contracts with native dependency links.

### When NOT to use
- The design is not yet approved and the user wants to explore approaches -> use `brainstorm`.
- The user wants a flat task breakdown without forge creation -> use `plan`.
- The user wants to implement an existing issue -> use `issue-autopilot`.
- The user wants git/commit operations -> use `scm`.

### Expected inputs
- `mode`: `create` or `update`.
- `design` (create): a design-doc path (approved brainstorm output) or an in-prompt description.
- `epic` (update): the epic number/URL, and the change description.
- `repos`: target repo list (user-provided) or the cwd repo.
- `forge`: detected from `git remote get-url origin`; ask if ambiguous.

### Expected outputs
- Forge-created epics + issues (full bodies, native `blocked_by` links, annotation pass complete).
- Updated epic + issues (update mode) with a delta map shown for approval.
- URLs reported; dry-run artifacts + state file in `.agents/results/epic-forge/<run>/`.

### State file format (`.agents/results/epic-forge/<run>/state.json`)
```json
{
  "run_id": "<run>",
  "forge": "github|gitlab",
  "repos": ["owner/repo", "..."],
  "epics": [{"repo": "owner/repo", "prefix": "D", "title": "...", "issue_number": 26}],
  "issues": [{"prefix": "D-1", "repo": "owner/repo", "title": "...", "issue_number": 27, "status": "created|linked|annotated"}],
  "links_created": [{"from": "D-5", "to": "D-2", "forge_id": 12345}]
}
```
The state file is written after each create/link/annotate step and drives resume + the
annotation pass (prefix → number registry). `gitignore` it via the `.agents` library
convention (run-scoped artifacts).

### Dependencies
- `_shared/runtime/providers.md` — forge CLI map (extended with create/link/enumerate/linked-PR ops).
- `_shared/core/clarification-protocol.md` — when to ask.
- `brainstorm` + `plan` skills — decomposition engine (referenced, not reimplemented).
- `skill-creator` — SSL-lite format + validation checklist.
- Local resources: `resources/epic-template.md`, `resources/issue-template.md`, `resources/style-guide.md`, `resources/examples/`.

### Control-flow features
- Branches by mode (create vs update).
- Branches by design approval state (brainstorm gate).
- Branches by change complexity (small tweak / missing feature / spec divergence).
- Two approval gates in create mode (DAG review, then summary review).
- One approval gate in update mode (delta map + per-issue delta).
- Any ambiguity -> `question` tool; never assume.

## Structural Flow

### Entry
1. **Clarification gate (MANDATORY).** If any of mode / design / epic / change / repos / forge is missing or ambiguous, ask via the `question` tool — one question at a time. Do NOT proceed on assumptions.
   - Ask mode: create or update?
   - Update: which epic (number/URL), what change?
   - Create: what are we creating (design-doc path or description)?
   - Repos: which repos (or use the cwd repo)?
   - Forge: detect from `git remote get-url origin`; if ambiguous, ask.
2. Detect provider per `providers.md`; verify auth with a functional request against the repo (`gh repo view` / `glab repo view`). Abort if it fails.
3. Resolve repos: user list or cwd repo.

### Scenes

#### Create mode

1. **PREPARE**: Load the design. If it is not an approved design (no design doc / vague description), run the `brainstorm` phase to produce one (save to `docs/plans/designs/`). Confirm the design with the user before decomposing.
2. **ACQUIRE**: Read the design; detect existing issue prefixes in the target repos (scan open + closed issues) to propose a non-colliding per-epic prefix; ask if ambiguous.
3. **REASON**: Decompose using `plan`-skill heuristics — waves, dependencies, critical path, cross-repo gates. Priority ladder is per epic-forge (`resources/style-guide.md`): P0 = critical-path foundation; P1 = high; P2 = medium; P3 = polish. Produce the epic + task DAG.
4. **Gate 1 (DAG review)**: present the DAG (prefix-based) for approval. On rejection, revise.
5. **ACT (draft)**: Write epic + issue bodies from the templates (`resources/epic-template.md`, `resources/issue-template.md`) following `resources/style-guide.md`. Bodies use prefix references + `{SIBLING_EPIC}` placeholders. Save all bodies + a manifest (prefix → title → phase → priority → deps → status) + an empty state file to `.agents/results/epic-forge/<run>/`.
6. **Gate 2 (summary review)**: present per-epic summary — issue cards (title, 2–3 line summary, files, acceptance count, depends-on) + numbered dependency hierarchy + critical path. **Verify write access to all target repos** (functional check per repo; fail fast). On approval, proceed.
7. **ACT (create + link + annotate)** — three passes:
   - Pass 1: create epics per `providers.md` "Epic representation & creation" (GitHub: issue + `epic` label; GitLab: native epic or issue+label fallback) — get numbers, then create issues (bodies reference `Epic: #N`). Record prefix → number in the state file. Pace calls; on 429 back off and resume from the state file; idempotency guard (skip already-created by title).
   - Pass 2: link native `blocked_by` per `providers.md` (GitHub database ids; cross-repo supported; markdown-hyperlink fallback if the forge rejects).
   - Pass 3: **annotation pass** — update every body in place, replacing: every prefix reference (epic task table, issue "Depends on"/"Blocked by", sibling-epic references, cross-repo prefixes), the `{SIBLING_EPIC}` placeholder (→ `repo#number`), and any `#{epic-number}` placeholder (→ the epic's number), each with the actual issue number + hyperlink.
8. **VERIFY**: confirm all issues created, all links present, all bodies annotated. Report URLs.

#### Update mode

1. **PREPARE**: Load the epic (number/URL) + change description. Confirm understanding of the request with the user if ambiguous.
2. **ACQUIRE**: Read the epic body. **Enumerate its issues via forge state** (`providers.md` enumerate op) — never the task table. Read each issue's state + close reason (GitHub `stateReason`; GitLab has no close-reason field — infer from labels/closed-by, ask if ambiguous) + linked PRs/commits (best-effort) to determine done vs not-done vs ambiguous (ask on ambiguity). **Follow companion epics/issues** in other repos (sibling references + cross-repo `blocked_by` links).
3. **REASON (delta map)**: Map the change request onto the issue set. Complexity-branch:
   - Small tweak → no brainstorm; plan the edit.
   - Missing feature → full `brainstorm` + `plan` for the new work.
   - **Spec divergence → the user must describe how and why** (never auto-detect); plan the spec-follows-code update from that description.
   Use targeted questions (clarification-protocol) to resolve ambiguity. Produce the delta map: affected open issues, obsolete closed issues, new issues needed, cross-repo companion impacts.
4. **ACT (draft)**: Draft updates — update the epic's spec/narrative sections (the contract), create new issue bodies, update open issue bodies, plan re-links. Refresh the epic task table as reference only (add new rows, update DAG/critical path) — do NOT maintain its status column.
5. **Gate (delta review)**: present the delta map + per-issue summary cards + **per-issue delta changes** (what is changing in each issue). On approval, proceed.
6. **ACT (apply)**: update the epic, create new issues, update open issue bodies, close obsolete issues (with user confirmation per closure), re-link dependencies, run the annotation pass for new references.
7. **VERIFY**: confirm the epic + issues reflect the delta. Report.

### Transitions
- If the design is not approved -> run `brainstorm` first (create mode).
- If a change is a missing feature -> full brainstorm + plan (update mode).
- If a change is spec divergence -> user must describe it; no auto-detection.
- If the forge rejects a cross-repo link -> fall back to markdown hyperlinks.
- If an issue's done-state is ambiguous -> ask.
- If creation fails partway -> resume from the state file; report what was created.

### Failure and recovery
| Failure | Recovery |
|---------|----------|
| Partial creation (rate limit / network / auth) | Resume from the state file (skip already-created, continue); report what was created |
| 429 / secondary rate limit | Back off (exponential), pause, resume from state file |
| No write access to a target repo | Detected at Gate 2 (fail fast); report which repo + what's needed |
| Forge rejects a cross-repo link | Fall back to markdown hyperlinks |
| Epic's issues can't be enumerated | Surface the discrepancy (search vs task table) and ask |
| Ambiguous done-state / delta mapping | Ask via `question` tool |
| Duplicate creation on re-run | Idempotency guard (skip by title) |

### Exit
- Success: epics + issues created/linked/annotated (or updated), URLs reported, state file + artifacts saved.
- Partial success: what was created vs not, remaining blockers explicit.
- Failure: blocking ambiguity or access problem explicit; nothing half-created left un-reported.

## Logical Operations

### Actions
| Action | SSL primitive | Evidence |
|--------|---------------|----------|
| Detect forge / auth | `READ` | `providers.md`; `git remote get-url origin`; `gh/glab repo view` |
| Clarify mode/context | `REQUEST` | `question` tool |
| Load design / approve | `READ` / `VALIDATE` | design doc or description; brainstorm gate |
| Decompose to DAG | `INFER` | `plan` heuristics; waves/deps/priority/critical path |
| Draft bodies | `WRITE` | templates + style-guide; `.agents/results/epic-forge/<run>/` |
| Review gates | `VALIDATE` | DAG review; summary review; delta review |
| Create / link / annotate | `CALL_TOOL` | `providers.md` forge ops; three passes |
| Report | `NOTIFY` | URLs + state file |

### Tools and instruments
- `gh` / `glab` (per `providers.md`).
- `question` tool (all user interaction).
- `task` tool (subagents for review/qa/refine when executing via ultrawork).
- Local resources: templates, style-guide, examples.

### Canonical workflow path
1. Clarify mode + context (ask if missing).
2. Detect forge + verify auth + resolve repos.
3. Create: load/approve design → decompose → DAG gate → draft → summary gate → create+link+annotate → report.
4. Update: read epic + enumerate issues (forge state) + companion epics → delta map (complexity-branch) → draft → delta gate → apply → report.

### Resource scope
| Scope | Resource target |
|-------|-----------------|
| `CODEBASE` | Target repos (issues/epics only — no source edits) |
| `LOCAL_FS` | `.agents/results/epic-forge/<run>/` (bodies, manifest, state file); `docs/plans/designs/` (approved designs) |
| `PROCESS` | `gh` / `glab` CLI calls (create, link, enumerate, view) |

### Preconditions
- User has write access to all target repos (verified at Gate 2).
- Forge CLI authenticated (functional check).
- Design approved (or brainstorm gate run).

### Effects and side effects
- Forge issues/epics created, linked, and annotated (network calls).
- `.agents/results/epic-forge/<run>/` artifacts written (bodies, manifest, state file).
- No source code is modified.
- Update mode may close issues (each closure confirmed with the user).

### Guardrails
1. **No-assumption rule (MANDATORY):** any ambiguity → ask via the `question` tool. Never make an assumption the user should answer. Follow `_shared/core/clarification-protocol.md` (LOW → proceed, MEDIUM → present options, HIGH → ask immediately).
2. **No local-docs/gitignored references:** all detail inline in epic/issue bodies; never cite `docs/`, `.agents/`, or gitignored files as the source of detail.
3. **Completion = forge state**, never the epic task table.
4. **Spec divergence is never auto-detected** — the user must describe it.
5. **No source edits** — the skill only creates/updates forge issues and local artifacts.
6. **Dry-run artifacts before any forge write** (create mode) so nothing is lost on failure.

## References
- Epic template: `.agents/skills/epic-forge/resources/epic-template.md`
- Issue template: `.agents/skills/epic-forge/resources/issue-template.md`
- Style guide: `.agents/skills/epic-forge/resources/style-guide.md`
- Worked example: `.agents/skills/epic-forge/resources/examples/README.md`
- Forge CLI map: `.agents/skills/_shared/runtime/providers.md`
- Clarification protocol: `.agents/skills/_shared/core/clarification-protocol.md`
- Design skill: `.agents/skills/brainstorm/SKILL.md`
- Plan skill: `.agents/skills/plan/SKILL.md`
- Skill format + validation: `.agents/skills/skill-creator/SKILL.md`
