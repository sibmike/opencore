# Doc architecture for a full-stack-web-aws project

A FastAPI + Next.js + AWS project needs a layered doc set that separates four concerns: the tech contract (what the system is), the behavioral contract (how agents and developers work inside it), the lifecycle contract (how features and docs evolve), and the quality contract (what "done" means). Each layer has a defined owner, a defined trigger for updates, and a defined relationship to every other layer. Without that wiring, docs accumulate, contradict, and stop being read.

---

## The doc set

Maintain the following files at minimum. Each has a single-sentence purpose that must remain stable even as content evolves.

| Doc | Single-sentence purpose |
|---|---|
| `README.md` | GitHub landing page and new-hire orientation; tech stack, architecture diagram, project structure, quick-start commands, infra table, milestones. |
| `backend/README.md` | FastAPI developer reference: route table, test stats, coverage, setup commands. |
| `frontend/README.md` | Next.js developer reference: hooks inventory, MSW handler count, test category breakdown, setup commands. |
| `docs/agent-behavior.md` | Behavioral contract: workflow rules, verification gates, feature lifecycle, deployment planning discipline. |
| `docs/architecture.md` | System design: component boundaries, data flow, Cognito + auth wiring, tech-stack table. |
| `docs/conventions.md` | Code style + naming: Python/Pydantic/aioboto3 patterns, TypeScript/TanStack Query patterns, anti-patterns. |
| `docs/debugging.md` | Debugging protocol (Observe → Hypothesize → Verify → Plan → Implement) + deployment-specific checklists. |
| `docs/harness-guidelines.md` | Meta-rules: what each doc is for, feedback loops, critical-file protection, entropy countermeasures. |
| `docs/workflows.md` | Git workflow, merge procedure, doc-maintenance cadence (per-commit, per-milestone), plan-file discipline. |
| `docs/infrastructure.md` | AWS resource inventory: CloudFormation stacks, DynamoDB tables, Aurora config, env vars, IAM roles. |
| `docs/deployment.md` | Step-by-step deploy: CloudFormation ordered deploy, App Runner setup, Amplify setup, env var split, custom domains, post-deploy smoke test. |
| `docs/data-model.md` | DynamoDB access patterns, Aurora schema, versioned config pattern, transient deletion fields. |
| `docs/security.md` | Auth model, secrets inventory, Cognito configuration, known security gaps. |
| `docs/testing.md` | Testing strategy: pytest + moto + aioboto3 patterns, Vitest + MSW v2, Playwright E2E setup, coverage floor. |
| `docs/ui-review-checklist.md` | Mandatory pre-commit Tailwind audit: desktop 1280px + mobile 375px, 10 bad-pattern fixes. |
| `docs/post-deploy-checklist.md` | Post-deploy gate: infra footprint check, smoke test, doc update procedure, memory check. |
| `docs/functionality-audit.md` | Living audit: implemented features, known gaps (permanent sequential numbers), pre-launch blockers, suggestions. |
| `docs/feature-proposals.md` | Strategic (P-numbered) and tactical (S-numbered) proposals; append-only Decision Log. |
| `docs/quality-score.md` | Per-domain quality grades (A–F) with pre-launch blocker tracking and post-launch priority queue. |
| `docs/core-beliefs.md` | Engineering values as small numbered directives; rarely changed; philosophy, not implementation. |
| `docs/exec-plans/active/` | Active execution plans for in-flight features; move to `completed/` on ship. |
| `docs/exec-plans/completed/` | Shipped execution plans; preserved as versioned history. |
| `docs/design-docs/` | Per-feature architectural decision records. |
| `docs/product-specs/` | Per-feature product requirements. |
| `ERRATA.md` | Permanent record of severe mistakes; each entry carries a numbered ID; read before any similar operation. |
| `MEMORY.md` (machine-local, not git-tracked) | Machine-specific paths, pitfalls, and environment discoveries; never for rules (rules go in committed docs). |

The three READMEs serve distinct audiences. `README.md` is GitHub-first: assume a reader who has never run the project. `backend/README.md` assumes a FastAPI developer with the venv already active. `frontend/README.md` assumes a Next.js developer with Node installed. Never list a planned feature as implemented in any README; counts must match test-suite output, not aspirations.

---

## Cross-doc wiring

Each doc type depends on others in predictable ways. Know the dependency graph to avoid circular updates and update-storms.

**Entry-point doc → everything else.** The agent entry-point file (e.g., `CLAUDE.md`) defines the session start read order and points at every other doc by path. It does not duplicate content from the docs it references. Its session start section lists files in strict read order: agent-behavior → harness-guidelines → workflows → conventions → core-beliefs → ERRATA → machine-local MEMORY → active exec plans.

**`agent-behavior.md` → `harness-guidelines.md`.** Agent behavior holds the behavioral contract (what to do). Harness guidelines holds the meta-contract (why the doc structure exists, how to change it, what "critical file" means). Never merge them — agent behavior drifts fast, harness guidelines should be stable.

**`architecture.md` → `infrastructure.md` → `deployment.md`.** Architecture describes component relationships. Infrastructure enumerates concrete AWS resources. Deployment gives the step-by-step execution. A change that touches all three (e.g., adding a new DynamoDB table with a GSI) must update all three. Infrastructure is the single source of truth for resource names; deployment references them by name, never by ARN or inline string.

**`functionality-audit.md` → `feature-proposals.md`.** The audit tracks what exists and what gaps remain (numbered permanently). The proposals track strategic decisions about what to build next. Proposals graduate to exec plans; they never move back to the audit until code is committed and tested.

**`exec-plans/active/` → `design-docs/` + `product-specs/`.** Every exec plan must cite a product spec and a design doc. If either is missing, create them before the plan is approved. On ship, move the exec plan to `completed/`; update the product spec to mark the feature complete; update the design doc if the implementation deviated.

**`post-deploy-checklist.md` → everything.** The post-deploy checklist is the reconciliation point where all other docs get updated. A feature deploy triggers: exec plan → completed; plan file renamed; product spec updated; design doc updated; functionality audit updated; infrastructure doc updated; testing doc updated; entry-point build status updated; all three READMEs updated. A bug fix triggers: entry-point build status; debugging Decision Record; functionality audit gaps; infrastructure doc if infra changed; ERRATA if the mistake was severe enough to warrant a permanent ID.

**`ERRATA.md` → `debugging.md` + `agent-behavior.md`.** An ERRATA entry is written when a mistake is severe enough that it must never be repeated. The entry carries a numbered ID, a root-cause diagnosis, and a prevention check. Reference the ERRATA ID in the commit message, exec plan, and any doc that describes the corrected pattern. Never delete an ERRATA entry — corrections compound its value.

---

## Plan-file discipline

Execution plans are first-class artifacts, not scratch notes. Treat them with the same discipline as code.

Store active plans in `docs/exec-plans/active/` and completed plans in `docs/exec-plans/completed/`. Each exec plan has a 1:1 paired machine-local plan file at `~/.claude/plans/<kebab-name>.md`. The kebab name must match the exec-plan filename exactly — auto-generated session titles are wrong names. For multi-phase work, append to the existing plan file; never create a parallel second plan for the same feature.

At session start, rename the plan file to the correct kebab name on approval. At session end, rewrite the plan as an execution record: what was planned, what was done, what deviated, what deferred work remains.

When you receive plan-mode approval, the plan must document:
- Every AWS surface touched (DynamoDB tables/GSIs, Aurora columns, IAM policies, CloudFormation resources, env vars, Cognito config, Lambda/Step Functions/SQS/SNS, App Runner/Amplify config changes).
- Delta vs. baseline — even when delta is zero, record it explicitly.
- Rollback path if the plan is rejected mid-execution.

Plans that skip the infra-footprint section have caused "table does not exist" production failures. The check is not optional.

---

## Doc-maintenance cadence

Docs go stale in proportion to commit velocity. Enforce cadence at two granularities.

**Per-commit.** Update `functionality-audit.md` if any functionality changed. Update quality scores if grades were affected. Update test counts in the entry-point build status table. These three updates take under two minutes; skipping them is the primary cause of audit drift.

**Per-milestone.** Run a full audit loop: functionality audit → quality scores → tech-debt tracker → core-beliefs review (did your team's philosophy evolve?) → move completed exec plans from `active/` to `completed/` → update all three READMEs. The three-README update is mandatory on every milestone ship — route tables, test counts, and coverage percentages all diverge within one milestone if not synchronized.

**What to update when (lookup table):**

| Trigger | Files to update |
|---|---|
| New DynamoDB table | `infrastructure.md` + verify in AWS Console |
| New API endpoint | `functionality-audit.md` + `backend/README.md` |
| New CloudFormation resource | `infrastructure.md` + `deployment.md` + verify in AWS |
| New Aurora column + Alembic migration | `data-model.md` + `infrastructure.md` |
| Bug fix | `debugging.md` Decision Record + ERRATA if severe |
| Feature ship | All six: exec plan → completed; product spec; design doc; functionality audit; infrastructure; testing |
| Security gap resolved | `security.md` gap table + `functionality-audit.md` |

Never update MEMORY.md with conventions or rules. MEMORY.md is for machine-specific pitfalls (a conda PATH collision on Windows, a broken symlink in a specific worktree) and environment discoveries (a DynamoDB Local port conflict). Conventions belong in `docs/conventions.md`. Debugging patterns belong in `docs/debugging.md`. Deployment steps belong in `docs/deployment.md`. Rules belong in `docs/agent-behavior.md`.

---

## Functionality audit discipline

The functionality audit is a living document, not a release note. It tracks four things simultaneously: what is implemented, what is planned, what gaps exist, and what suggestions have been made.

Gap numbers are permanent and sequential (#1, #2, …) and never reused. When a gap is resolved, mark it resolved with a note — do not delete it. Severity is High / Medium / Low. The pre-launch blockers list is a subset of the gap list: it must reach zero before any production launch.

Pre-launch blockers for a FastAPI + Next.js + AWS project typically concentrate in five categories:
1. **Rate limiting** — without `slowapi` middleware on Bedrock-invoking endpoints, compute costs are unbounded.
2. **Seed data** — required DynamoDB lookup tables that are empty on first deploy cause silent failures (a CORS error is often the first symptom — it is not actually CORS).
3. **AI score aggregation wiring** — end-to-end path from data-write to Bedrock invocation to result storage to frontend render must be smoke-tested, not just unit-tested.
4. **Auth completeness** — password reset flow and sign-out UI are blocking even if Cognito backs them.
5. **Sign-out surface** — a session that cannot be ended is a security blocker.

Post-launch priority order: rate limiting → notification system (WebSocket or polling + DynamoDB notifications table) → multi-resource switching → bulk upload → custom domain in CloudFormation → CloudWatch alarms (5xx, latency, cost) → per-resource analytics.

---

## Quality tracking

Maintain a `docs/quality-score.md` with a grade (A/B/C/D/F) per domain: Backend, Frontend, Infrastructure, Testing, Documentation. Update after every grading pass.

Track pre-launch blockers as domains that must reach B or higher before ship. Mark resolved entries with a RESOLVED note and the resulting grade; use strikethrough to preserve historical record. Do not delete entries — the grading history is a record of what was improved and why.

Track a post-launch priority queue as an ordered list of domains or subsystems that need grade improvement after launch. Struck-through entries indicate resolved items; new entries are appended, not inserted.

The quality doc is not a vanity dashboard. It is the forcing function for post-launch technical debt prioritization. If a domain sits at D for two milestones without a plan entry, that is a process failure, not just a quality gap.

---

## Generated documentation

Three categories of docs can be generated automatically — generate them and mark them as generated.

1. **API catalog** — generated from FastAPI routers and Pydantic schemas. Each generated file carries a `GENERATED — do not edit manually; regenerate with <command>` header.
2. **Component inventory** — generated from the Next.js `components/` directory. Same header convention.
3. **Database schema** — generated from DynamoDB CloudFormation YAML and Alembic migration files. For DynamoDB, the source of truth is the CloudFormation template; for Aurora, the source of truth is the latest Alembic head.

Never maintain generated docs by hand. When the source changes, regenerate. When regeneration breaks, fix the generator, not the output file.

---

## Critical-file protection and entropy countermeasures

Two files anchor the entire doc system and must be git-tracked from project initialization: the agent entry-point file and ERRATA.md. If either is lost, the session start ritual breaks and mistake records are destroyed.

For gitignored files — particularly MEMORY.md and machine-local plan files — never delete, move, or overwrite without confirmation and a backup. These files are not in git; they cannot be recovered from history.

Run entropy countermeasures on a regular cadence:
- `npm audit` (frontend dependencies)
- `pip-audit` (backend dependencies)
- Grep for convention violations (sync `boto3` calls, `HTTPException` raised from service layer, data fetching outside TanStack Query)
- Remove dead code, rotate stale dependencies

Structural constraints to enforce once CI is in place: services cannot import from routers; Pydantic validation must exist at every API boundary; import direction is enforced by a lint rule. Until the lint rule exists, grep for the violation pattern in every milestone review.

---

## The architecture doc itself

`docs/architecture.md` serves as the canonical system overview. It must contain:

**Tech-stack table.** Frontend: Next.js App Router + TypeScript + shadcn/ui + TanStack Query + Amplify v6 + Tailwind v4. Backend: Python + FastAPI + Pydantic v2 + aioboto3 + asyncpg + Alembic. AWS services: DynamoDB + Aurora Serverless v2 + S3 + Bedrock + SES + Lambda + Step Functions + App Runner + Amplify + Cognito + Route 53 + CloudFormation.

**Data flow diagram.** Next.js → FastAPI → DynamoDB (content + events). FastAPI → Aurora Serverless v2 (relational permissions). FastAPI → Lambda (async jobs). Cognito (auth + JWT). S3 + Bedrock (storage + AI).

**Auth + ownership model.** `get_current_user()` Depends() on every protected route. `get_owned_resource()` GSI-based ownership guard. `require_any_role()` factory for multi-role endpoints. `get_db()` yields aioboto3 resource via `Depends()`. This wiring must be documented here, not only in `security.md`, because it affects every endpoint's design.

**Monorepo layout.** `backend/` (FastAPI application + Alembic migrations + tests). `frontend/` (Next.js App Router + Vitest tests + Playwright E2E). `infra/` (CloudFormation templates + deploy scripts). `docs/` (all doc files). `.github/workflows/` (CI/CD). Worktree note: Next.js cannot run from a git worktree (node_modules absent, symlinking fails on Windows); backend Python tests can run from a worktree using the absolute venv path.

---

## Behavioral integrity: what the doc system prevents

The doc structure described here is not organizational aesthetics. Each element prevents a specific class of failure.

**Half-built features ship without a functional path.** Every mutation must have a visible `onError` toast. Every cross-store operation (DynamoDB + Aurora + Cognito + SES) belongs in a dedicated `<feature>_service.py`. No domain service imports from another. Audit label↔handler identifier parity before shipping. The functionality audit's pre-launch blocker list is the forcing function.

**Admin panel patterns that scale.** Paginated scan endpoints with BatchGetItem enrichment for listing. Two-step CSV uploads (Preview → Commit) with per-row partial success. Race-safe `ConditionExpression` upsert using `attribute_exists` guard on hot tables. Domain-derived admin auth (`email_verified` + email domain check) rather than a hardcoded allow-list.

**Debug discipline.** Debugging is systematic narrowing (Observe → Hypothesize → Verify → Plan → Implement), not trial-and-error. The `docs/debugging.md` Decision Record section carries Date / Issue / Root Cause / Fix / Lesson. Every severe mistake gets an ERRATA ID and a prevention check. The ERRATA ID propagates to the commit message, exec plan, and relevant docs so future readers can trace the lineage.

**Engineering judgment, not just rules.** `docs/core-beliefs.md` holds engineering values as small numbered directives — rarely changed, never cluttered with implementation detail. The harness guidelines hold the meta-contract for why the docs exist and how to change them. The two files are separate because philosophy should be stable while process must be malleable.

---

## Cross-references

- See [`../practices/conventions-fullstack.md`](../practices/conventions-fullstack.md) for Python/TypeScript code conventions, TanStack Query patterns, naming, and anti-patterns.
- See [`../practices/deployment-aws.md`](../practices/deployment-aws.md) for App Runner / Amplify deploy procedures, env var split, CloudFormation ordered deploy, and post-deploy smoke test.
- See [`../practices/infra-aws.md`](../practices/infra-aws.md) for DynamoDB multi-table patterns, Aurora Serverless v2, Bedrock invocation, S3/SES service patterns.
- See [`../practices/security-aws.md`](../practices/security-aws.md) for Cognito JWT Depends() factory, pre-signed URL discipline, Google SSO wiring, secrets inventory.
- See [`../practices/testing-fullstack.md`](../practices/testing-fullstack.md) for pytest + moto + aioboto3 async patterns, Vitest + MSW v2, Playwright E2E, coverage conventions.
- See [`../practices/debugging-fullstack.md`](../practices/debugging-fullstack.md) for frontend-backend wiring failures, CORS vs. missing table diagnostics, App Runner health-check diagnostics.
- See [`../practices/ui-review-frontend.md`](../practices/ui-review-frontend.md) for Tailwind responsive audit protocol, 10 bad-pattern fixes, pre-commit verification script.
- See [`../practices/dev-environment-fullstack.md`](../practices/dev-environment-fullstack.md) for Python venv + Node interop, Windows PATH pitfalls, worktree constraints.
- See [`../../../../core/practices/session-protocol.md`](../../../../core/practices/session-protocol.md) for agent session ritual and behavior-defining file read order.
- See [`../../../../core/practices/feature-lifecycle.md`](../../../../core/practices/feature-lifecycle.md) for exec-plan, design-doc, and product-spec lifecycle.
- See [`../../../../core/practices/git-workflow.md`](../../../../core/practices/git-workflow.md) for dev→main branch model, commit discipline, hotfix flow.
- See [`../../../../core/practices/infra-management.md`](../../../../core/practices/infra-management.md) for plan-mode approval gates and net infra footprint check.
- See [`../../../../core/documentation/doc-maintenance.md`](../../../../core/documentation/doc-maintenance.md) for generic doc cadence and inheritance markers.

---

<!-- Assembled from: agent-behavior.md §"1. Workflow Orchestration", agent-behavior.md §"9. Feature Development Lifecycle", architecture.md §"Tech Stack", CLAUDE.md §"Session Start (MANDATORY — read on EVERY new session)", CLAUDE.md §"2. Git commit discipline: dev → main", CLAUDE.md §"Documentation Map", ERRATA.md §"ERR-009: Half-Built Feature Shipped Without Email Path or Toast — UI Gave No Signal of Failure", functionality-audit.md §"1.30 Basic Admin Panel (M15)", functionality-audit.md §"Pre-Launch Blockers (Must Fix Before Production)", functionality-audit.md §"Post-Launch Priority Queue", harness-guidelines.md §"2.8 Plans as First-Class Artifacts", harness-guidelines.md §"2.9 Critical Files Are Sacred", harness-guidelines.md §"2.10 Encode Judgment, Not Just Rules", harness-guidelines.md §"2.11 Debug Like an Engineer, Not a Gambler", harness-guidelines.md §"4.5 README Files Role", harness-guidelines.md §"4.6 MEMORY.md Role", harness-guidelines.md §"4.7 Agent Behavior Doc Role", harness-guidelines.md §"5.2 Deterministic Constraints (Currently Active)", harness-guidelines.md §"5.3 Structural Constraints (To Build)", harness-guidelines.md §"6.2 Countermeasures", harness-guidelines.md §"6.4 Golden Principles", harness-guidelines.md §"8. Generated Documentation", post-deploy-checklist.md §"3. Documentation Updates — Feature Deploys", post-deploy-checklist.md §"4. Documentation Updates — Bug Fixes / Hotfixes", post-deploy-checklist.md §"6. Memory Check", post-deploy-checklist.md §"Quick Reference: What to Update When", quality-score.md §"Summary", quality-score.md §"Pre-Launch Blockers (must reach B or higher)", quality-score.md §"Priority Queue (post-launch)", README.md §"## Architecture", README.md §"## Tech Stack", README.md §"## Project Structure", workflows.md §"Functionality Doc Maintenance", workflows.md §"Feature Proposals Doc", workflows.md §"Per-Commit", workflows.md §"Per-Milestone", workflows.md §"Plan File Management", workflows.md §"README Maintenance" -->
