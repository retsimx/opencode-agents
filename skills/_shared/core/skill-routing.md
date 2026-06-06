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
| review, security, performance | **qa** | |
| accessibility, WCAG, a11y | **qa** | |
| UI design, design system, landing page, DESIGN.md, color palette, typography, glassmorphism, responsive design | **design** | |
| brainstorm, ideate, design, explore, idea, concept | **brainstorm** | Run before pm |
| plan, breakdown, task, sprint | **pm** | |
| automatic, parallel, orchestrate | **orchestrate** | |
| workflow, guide, manual, step-by-step | **coordination** | |
| configuration management, SCM, CM, git, commit, gitflow, GitHub Flow, GitLab Flow, trunk-based branching, merge conflict, rebase, worktree, baseline, tag, release branch, signed commits, merge queue, conventional commits | **scm** | SCM + Conventional Commits in one skill |

---

## Complex Request Routing

| Request Pattern | Execution Order |
|----------------|-----------------|
| "Create a fullstack app" | pm → (backend + frontend) parallel → qa |
| "Create a mobile app" | pm → (backend + mobile) parallel → qa |
| "Fullstack + mobile" | pm → (backend + frontend + mobile) parallel → qa |
| "Help me choose the system architecture" | architecture → pm |
| "Review this architecture before we build" | architecture → pm → qa |
| "Fix bug and review" | debug → qa |
| "Add feature and test" | pm → relevant agent → qa |
| "I have an idea for a feature" | brainstorm → pm → relevant agents → qa |
| "Let's design something new" | brainstorm → pm → relevant agents → qa |
| "Do everything automatically" | orchestrate (internally pm → agents → qa) |
| "I'll manage manually" | coordination |
| "Design and build a landing page" | design → frontend |
| "Design, build, and review" | design → frontend → qa |
| "Redesign based on this URL" | design (Phase 2 EXTRACT) → frontend |

---

## Inter-Agent Dependency Rules

### Parallel Execution Possible (No Dependencies)
- backend + frontend (when API contract is pre-defined)
- backend + mobile (when API contract is pre-defined)
- frontend + mobile (independent of each other)

### Sequential Execution Required
- architecture → pm (architecture decision comes before task decomposition)
- brainstorm → pm (design comes before planning)
- pm → all other agents (planning comes first)
- implementation agent → qa (review after implementation complete)
- implementation agent → debug (debugging after implementation complete)
- backend → frontend/mobile (when executing parallel without API contract)

### QA Is Always Last
- qa runs after all implementation tasks are complete
- Exception: Can run immediately if user requests review of specific files only

---

## Escalation Rules

| Situation | Escalation Target |
|-----------|------------------|
| Agent finds bug in different domain | Create task for debug |
| QA finds CRITICAL issue | Re-run relevant domain agent |
| Architecture change needed | architecture → pm |
| Performance issue found (during implementation) | Current agent fixes, debug if severe |
| API contract mismatch | orchestrate re-runs backend |

---

## Turn Limit Guide by Agent

| Agent | Default Turns | Max Turns (including retries) |
|-------|--------------|------------------------------|
| pm | 10 | 15 |
| backend | 20 | 30 |
| frontend | 20 | 30 |
| mobile | 20 | 30 |
| architecture | 12 | 18 |
| debug | 15 | 25 |
| qa | 15 | 20 |
