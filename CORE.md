# CORE — Engineering Practices

Versioning: semver. Current version: 0.1.2.

## Practices
- [anti-patterns](practices/anti-patterns.md) — Generalizable mistakes — state loss, infra drift, premature action, scope creep.
- [architecture-principles](practices/architecture-principles.md) — Positive design rules — purpose-fit storage, layering, typed exceptions, async polling.
- [debugging](practices/debugging.md) — Diagnostic protocol — observe, hypothesize, verify, plan, implement.
- [feature-lifecycle](practices/feature-lifecycle.md) — Doc trail from spec to design to exec plan to verification to archive.
- [git-workflow](practices/git-workflow.md) — Branching model, commit discipline, merge procedure.
- [infra-management](practices/infra-management.md) — Infrastructure change discipline; verify-after-deploy; deploy sequencing; CI/CD wiring.
- [plan-mode](practices/plan-mode.md) — When plan mode is mandatory and what makes a plan adequate.
- [security-principles](practices/security-principles.md) — Auth/authz layering, secrets, least-privilege defaults.
- [session-protocol](practices/session-protocol.md) — Session start/during/end behavior; subagent strategy; context discipline.
- [testing-principles](practices/testing-principles.md) — What tests prove and don't; coverage philosophy; integration vs unit.

## Documentation
- [checklist-protocol](documentation/checklist-protocol.md) — Authoring and maintaining checklists; quality-tracking artifacts.
- [doc-maintenance](documentation/doc-maintenance.md) — Update triggers, screenshot workflow, runbook structure, doc-trail cadence.
- [writing-style](documentation/writing-style.md) — Voice, density, structural conventions for human-facing docs.

## Evolution
- [dreaming](evolution/dreaming.md) — Per-session reflection prompt; when to run; staged-PR output discipline.
- [delta-extraction](evolution/delta-extraction.md) — Batch-process dreams every 5 dreams or weekly; aggregate, filter, produce PR bundles.
- [core-update-gate](evolution/core-update-gate.md) — Promotion criteria for upstream CORE PRs; ≥3 engineers from ≥2 projects.
- [project-core-divergence](evolution/project-core-divergence.md) — Monthly or end-of-project diff of forked practices vs upstream CORE.
- [stack-divergence](evolution/stack-divergence.md) — Monthly or end-of-project diff of forked stack-practices vs upstream stack module; ≥2 engineers from ≥2 projects of same archetype.
- [checklist-maintenance](evolution/checklist-maintenance.md) — When to add/remove checks; promotion of checks across projects.

## Stack Modules
Per-archetype layers that sit between CORE and PROJECT, capturing wisdom common to projects of the same technical stack. See [stack-modules/README.md](stack-modules/README.md) for the 4-layer model and archetype concept.

- [stack-modules/full-stack-web-aws/](stack-modules/full-stack-web-aws/) — Python or TypeScript backend + React/Next.js frontend + AWS services (v0.1.0, bootstrapped 2026-05-10).

## Templates
- [dream-report.md.template](templates/dream-report.md.template) — Output skeleton for the dream prompt.
- [delta-report.md.template](templates/delta-report.md.template) — Output skeleton for the delta extraction prompt.
- [user-preferences.md.template](templates/user-preferences.md.template) — Confidence-tagged user-preferences scaffold.
- [project-scaffold/](templates/project-scaffold/) — Starter docs for a new project: CLAUDE.md, ERRATA.md, post-deploy-checklist.md templates plus empty `docs/dreams/` and `docs/deltas/` placeholder dirs.

## Versioning policy
- Patch: typo fixes, formatting, clarifications that do not change meaning.
- Minor: new sections within existing docs, new examples, deepened guidance, additive scaffolding (new evolution helpers, new templates).
- Major: new top-level docs, removed docs, contradictory updates that change meaning of an existing rule.

## Lineage
v0.1.0 extracted from source project on 2026-05-10. v0.1.1 added dreaming-system infrastructure (evolution/ + templates/) on 2026-05-10. v0.1.2 added STACK layer (stack-modules/ + evolution/stack-divergence.md) on 2026-05-10. See CORE-CHANGELOG.md.
