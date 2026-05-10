# CORE Changelog

## v0.1.2 — 2026-05-10
Added STACK layer to the model. The 3-layer model (CORE → PROJECT → USER) leaked: project docs in source projects carry significant archetype-level wisdom (CORS debugging, framework testing, Windows Python-alias pitfalls, conda+npm interop) that is more specific than CORE but less specific than PROJECT. A new project forking CORE re-discovers all of it. STACK closes that gap.

- New top-level dir: `stack-modules/` with concept README and the first archetype scaffold `full-stack-web-aws/` (empty; v0.0.0).
- New evolution doc: `evolution/stack-divergence.md` — mirror of project-core-divergence.md at the stack layer; lower gate (≥2 engineers from ≥2 projects of the same archetype, vs CORE's ≥3 from ≥2 projects of any archetype).
- CORE.md now indexes Stack Modules section + Evolution gains stack-divergence row.
- The bootstrap extraction (one-time, from a source project's refactored docs into a stack module's v0.1.0) is documented as a separate orchestrator prompt at `Downloads/stack_extraction_prompt.md`.
- 4-layer model documented: CORE (universal) → STACK (per archetype) → PROJECT (codebase) → USER.

Versioning policy unchanged. Classified as Minor under the same "additive scaffolding" interpretation that justified v0.1.1.

## v0.1.1 — 2026-05-10
Added dreaming-system infrastructure. Additive scaffolding only; no edits to v0.1.0 practices or documentation. Classified as Minor under the v0.1.1-revised versioning policy ("additive scaffolding: new evolution helpers, new templates"). Strict v2 reading would call new top-level docs Major; the Minor classification reflects that evolution/ + templates/ were always part of the v2 spec's intended CORE layout — they finish v0.1.0 rather than evolve it.

- 5 evolution docs added: dreaming, delta-extraction, core-update-gate, project-core-divergence, checklist-maintenance
- 6 templates added: dream-report.md.template, delta-report.md.template, user-preferences.md.template, plus project-scaffold/ subset (CLAUDE.md.template, ERRATA.md.template, post-deploy-checklist.md.template) with empty docs/dreams/ and docs/deltas/ placeholder dirs
- Versioning policy clarified: Minor now explicitly covers additive scaffolding (new evolution helpers, new templates).
- CORE.md index updated with Evolution and Templates sections; lineage line extended.

## v0.1.0 — 2026-05-10
Initial extraction from source project (anonymized as "the source project").

- 13 CORE docs created via Phase 2 synthesis of 23 stage1 split reports.
- Practices: anti-patterns, architecture-principles, debugging, feature-lifecycle, git-workflow, infra-management, plan-mode, security-principles, session-protocol, testing-principles
- Documentation: checklist-protocol, doc-maintenance, writing-style
- Source project's docs refactored to inherit from CORE via `docs/core-practices/` fork.
