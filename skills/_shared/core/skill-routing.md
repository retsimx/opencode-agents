# Skill Routing Map

Routing rules for orchestrate and coordination to assign tasks to the correct agent.

## Progressive Disclosure

Skills use two-stage loading to optimize context usage:

1. **Stage 1 (always loaded)**: `name` and `description` from SKILL.md frontmatter
2. **Stage 2 (on explicit invocation)**: Full SKILL.md body loaded only when skill is explicitly requested via /command or agent skills field

Skills are explicitly loaded via /command invocation or agent skills field. Load full instructions only for explicitly requested skills.

---

## Skill → Agent Mapping

| Skill Domain | Primary Skill | Notes |
|----------------------|---------------|-------|
| API, endpoint, REST, GraphQL, database, migration | **backend** | |
| auth, JWT, login, register, password | **backend** | Auth UI task can also be created for frontend |
| UI, component, page, form, screen (web) | **frontend** | |
| style, Tailwind, responsive, CSS | **frontend** | |
| mobile, iOS, Android, Flutter, React Native, app | **mobile** | |
| offline, push notification, camera, GPS | **mobile** | |
| architecture, system design, software design, module boundary, service boundary, tradeoff, ADR, ATAM, CBAM, quality attribute | **architecture** | Consult before planning when the structure itself is undecided |
| bug, error, crash, broken, slow | **debug** | |
| review, security, performance | **review** | |
| accessibility, WCAG, a11y | **review** | |
| PR review, MR audit, forge review, PR against issue, branch against main, pull request audit | **forge-review** | Post-implementation PR/MR review with provider (GitHub/GitLab) integration and inline comment generation |
| deep review, regression analysis, intent verification, bounded diff analysis | **deep-review** | Pure analysis engine over bounded diff/commit/PR |
| UI design, design system, landing page, DESIGN.md, color palette, typography, glassmorphism, responsive design | **design** | |
| brainstorm, ideate, design, explore, idea, concept | **brainstorm** | Run before plan |
| plan, breakdown, task, sprint | **plan** | |
| epic, issue, decompose design to issues, create epics/issues on the forge, update epic | **epic-forge** | Consumes brainstorm/plan; forge creation + linking |
| automatic, parallel, orchestrate | **orchestrate** | |
| workflow, guide, manual, step-by-step | **coordination** | |
| configuration management, SCM, CM, git, commit, gitflow, GitHub Flow, GitLab Flow, trunk-based branching, merge conflict, rebase, worktree, baseline, tag, release branch, signed commits, merge queue, conventional commits | **scm** | SCM + Conventional Commits in one skill |

---

## Complex Request Routing

| Request Pattern | Execution Order |
|----------------|-----------------|
| "Create a fullstack app" | plan → (backend + frontend) parallel → review |
| "Create a mobile app" | plan → (backend + mobile) parallel → review |
| "Fullstack + mobile" | plan → (backend + frontend + mobile) parallel → review |
| "Help me choose the system architecture" | architecture → plan |
| "Review this architecture before we build" | architecture → plan → review |
| "Fix bug and review" | debug → review |
| "Add feature and test" | plan → relevant agent → review |
| "I have an idea for a feature" | brainstorm → plan → relevant agents → review |
| "Create epics/issues from this design" | brainstorm → plan → epic-forge |
| "Update epic #N with this change" | epic-forge (update mode; brainstorm/plan branch on complexity) |
| "Let's design something new" | brainstorm → plan → relevant agents → review |
| "Do everything automatically" | orchestrate (internally plan → agents → review) |
| "I'll manage manually" | coordination |
| "Design and build a landing page" | design → frontend |
| "Design, build, and review" | design → frontend → review |
| "Redesign based on this URL" | design (Phase 2 EXTRACT) → frontend |
| "forge-review #154" | forge-review (fetch PR diff & context → deep-review subagent → generate scorecard & comments) |
| "review PR #42" | forge-review (fetch PR diff → deep-review subagent → review report) |
| "audit MR !10" | forge-review (fetch GitLab MR diff → deep-review subagent → review report) |
| "review PR against issue" | forge-review (fetch issue context + PR diff → deep-review subagent → audit against requirements) |
| "review branch against main" | forge-review (diff branch against main → deep-review subagent → review report) |

---

## Review & QA Domain Boundaries & Cross-Routing

| Skill | Primary Purpose | Scope & Input Mode | Output / Deliverables | Mutates PR? |
|-------|-----------------|--------------------|-----------------------|-------------|
| **`review`** | Broad QA, OWASP Top 10 security, performance, accessibility (WCAG 2.1 AA), code quality, ISO/IEC 25010 framing | Local repo, staging diff, or full workspace | Full QA report (`result-review-*.md`), optional fix-verify loop | No |
| **`deep-review`** | Deterministic 9-dimension post-implementation analysis (correctness, regressions, state/data, UI, tests, dead code, security, perf, DRY) | Bounded diff, commit SHA, file list | Structured 9-dimension report (`result-deep-review-*.md`) | No |
| **`forge-review`** | End-to-end Forge PR/MR audit against issue requirements with provider (`gh`/`glab`) integration | Remote PR/MR number, URL, or branch | PR Scorecard, inline suggestion comments, review deliverable | No (read + comment only) |
| **`gardener-harvest`** | Automated sequential merge queue assessment and squash-merging of open PR batches | Batch of open PRs on GitHub/GitLab | Merge queue state (`pr-merge-queue.json`), summary table, squash-merges passing PRs | Yes (squash-merge only) |

### Boundary and Cross-Routing Rules:
1. **Forge PR/MR Context & Comments (`forge-review`)**:
   - Route when the user specifies a PR/MR number (e.g. `forge-review #154`, `review PR #42`, `audit MR !10`), asks to review a PR against an issue, or wants inline review comments generated for GitHub/GitLab.
   - Delegates diff inspection and deep regression analysis to `deep-review` subagents while orchestrating provider interactions (`gh`/`glab`), scorecard generation, and comment formatting.
2. **Local Codebase & Pre-Ship QA (`review`)**:
   - Route for general pre-commit or pre-ship checks across the local workspace, security audits, accessibility audits, or fix-verify loops with domain agents.
3. **Pure Bounded Scope Analysis (`deep-review`)**:
   - Route when analyzing a strict, explicit diff/commit/patch without forge provider interactions, or when invoked as an analysis subagent by `forge-review` or implementation workflows.
4. **Merge Automation & Queue Processing (`gardener-harvest`)**:
   - Route when batch processing open PRs for automated sequential squash-merging. Never used for single-PR detailed qualitative reviews or issue audits.

---

## Inter-Agent Dependency Rules

### Parallel Execution Possible (No Dependencies)
- backend + frontend (when API contract is pre-defined)
- backend + mobile (when API contract is pre-defined)
- frontend + mobile (independent of each other)

### Sequential Execution Required
- architecture → plan (architecture decision comes before task decomposition)
- brainstorm → plan (design comes before planning)
- plan → all other agents (planning comes first)
- implementation agent → review (review after implementation complete)
- implementation agent → debug (debugging after implementation complete)
- backend → frontend/mobile (when executing parallel without API contract)

### Review Is Always Last
- review runs after all implementation tasks are complete
- Exception: Can run immediately if user requests review of specific files only

---

## Escalation Rules

| Situation | Escalation Target |
|-----------|------------------|
| Agent finds bug in different domain | Create task for debug |
| Review finds CRITICAL issue | Re-run relevant domain agent |
| Architecture change needed | architecture → plan |
| Performance issue found (during implementation) | Current agent fixes, debug if severe |
| API contract mismatch | orchestrate re-runs backend |

---

## Turn Limit Guide by Agent

| Agent | Default Turns | Max Turns (including retries) |
|-------|--------------|------------------------------|
| plan | 10 | 15 |
| backend | 20 | 30 |
| frontend | 20 | 30 |
| mobile | 20 | 30 |
| architecture | 12 | 18 |
| debug | 15 | 25 |
| review | 15 | 20 |
| forge-review | 15 | 25 |
| deep-review | 10 | 15 |
