# Stack Divergence Review

A project's `docs/stack-practices/` is forked from upstream `core/stack-modules/<archetype>/` at a specific stack version. The fork drifts as the project hits real conditions the inherited stack pattern did not anticipate. The stack divergence review measures that drift, classifies it, and turns generalizable parts into upstream stack-module PRs.

This doc parallels [`project-core-divergence.md`](project-core-divergence.md) at one layer down. Where the core divergence review promotes universal patterns across all archetypes, the stack divergence review promotes archetype-internal patterns within one archetype.

## When to run

Monthly, or at end-of-project, whichever comes first. Same cadence as the core divergence review; both can run in the same pass since they consume the same delta history.

## Output

Two artifacts:

1. **STACK PR drafts** — one per generalizable evolution. Each PR follows the format below.
2. **Stack divergence report** — full diff with classifications. Lives at `<project>/docs/deltas/stack-divergence-<YYYY-MM>.md`.

## Read first (mandatory inputs)

- Upstream stack module: `core/stack-modules/<archetype>/practices/`, `documentation/`, `templates/`. Current canonical stack version.
- The project fork: `<project>/docs/stack-practices/`. The drifted version.
- Project lineage: `<project>/docs/LINEAGE.md` showing which stack version the project forked from.
- Delta reports: `<project>/docs/deltas/`. History of why drifts happened.
- Project-core divergence report from the same period (if any). A drift may be classified as core-promotable there; if so, it does NOT also promote to stack — universal beats archetype-specific.

## The prompt

```markdown
# Stack Divergence Review

You are comparing a project's forked stack-practices against the current
upstream stack module to identify generalizable evolution candidates within
the archetype.

## Inputs
- Upstream STACK: core/stack-modules/<archetype>/ (current canonical)
- PROJECT fork: <project>/docs/stack-practices/ (drifted version)
- Delta reports: <project>/docs/deltas/ (history of why drifts happened)
- Project-core divergence (same period): <project>/docs/deltas/divergence-<YYYY-MM>.md (if exists)

## Process

### 1. Diff
For each file in docs/stack-practices/, diff against the corresponding
stack-module file. Categorize each divergence:

| File | Divergence | Category | Evidence |
|------|-----------|----------|----------|
| debugging-fullstack.md | Added "verify CORS preflight returns 200 not 204" | Stack-generalizable? | 4 dreams, 1 ERRATA |
| testing-fullstack.md | Added pytest-asyncio fixture pattern for boto3 client | Stack-generalizable | ERR-NNN, 6+ sessions |
| deployment-aws.md | Added "App Runner needs 60s post-deploy before health checks" | Stack-specific drift | Only matters for App Runner; would not generalize to Lambda |

### 2. Classify
- **Project-specific drift:** Stays in project. Not promoted.
  (e.g., "check the audit-log table after writing to it" — only relevant if you have an audit-log table)
- **Stack-generalizable evolution:** Would improve any project in this archetype.
  (e.g., "verify CORS preflight returns 200 not 204" — universal within full-stack-web-aws)
- **Core-generalizable:** Would improve any project regardless of stack.
  Promote to CORE divergence review instead, NOT to stack. Universal beats archetype-specific.
- **Contested:** Drift that might generalize but evidence is thin within the archetype.
  Hold for data from another project of the same archetype.

### 3. Draft STACK PR
For each stack-generalizable evolution:

\`\`\`
FILE: core/stack-modules/<archetype>/practices/<file>
CURRENT: <what stack module says>
PROPOSED: <what the drift taught us>
SOURCE PROJECT: <project name>
ARCHETYPE: <archetype>
EVIDENCE: <which deltas/dreams/ERRATA support this>
SESSIONS: <count>
\`\`\`

### 4. PR Requirements
- ≥2 engineer approvals required.
- Approvers must be from at least 2 projects of the SAME archetype.
- Each approver answers: "Would this have helped MY project in this archetype?"
  If <2 say yes → hold, gather more evidence.
- This is a LOWER threshold than CORE (which requires ≥3 from ≥2 archetypes).
  Stack content is narrower; archetype-internal evidence is sufficient.

## Output
- STACK PR (ready for archetype-internal review)
- Stack divergence report (full diff with classifications)
- Items held for cross-project (same archetype) validation
```

## Classification heuristics

A drift may sit on the boundary between stack-generalizable and core-generalizable. Apply these tests:

- **The "swap the framework" test.** If the project tore down its framework (FastAPI → Express, Next.js → Remix) but kept the same stack archetype (still full-stack-web-aws), would the rule still apply? Yes → stack-generalizable. No → framework-specific drift (stays in project, or split the archetype).
- **The "swap the cloud" test.** If the project moved from AWS to GCP but kept the archetype shape (still full-stack-web-cloud), would the rule still apply? Yes → consider promoting to CORE (universal across cloud vendors). No → stays in this archetype's stack module.
- **The "swap the archetype" test.** If a CLI-tool project hit the same problem, would the rule help? Yes → CORE candidate. No → STACK.

A drift that fails the "swap the framework" test is not necessarily wrong — it may signal that the archetype needs splitting (e.g., `full-stack-web-aws-fastapi` vs `full-stack-web-aws-express`). Splitting is heavy; default to keeping one archetype until at least 3 projects show divergent framework-specific drift.

## Interaction with core divergence

Run the core divergence review first, then the stack divergence review. Reasons:

- Core promotion takes precedence. A drift classified as core-generalizable should NOT also propose a stack PR — that creates duplicate maintenance.
- The core review's classifications inform the stack review's inputs ("this drift was already promoted to CORE; skip it here").

A drift that the core review held as contested may be promotable to STACK in the meantime — same content, narrower gate. Note the relationship in the stack divergence report.

## Bootstrap (one-time, before first divergence review)

A project that forks CORE but no matching stack module exists yet has no upstream to diff against. Before the first stack divergence review can run, the bootstrap extraction must produce v0.1.0 of the archetype. See `Downloads/stack_extraction_prompt.md` for the bootstrap orchestrator prompt.

The bootstrap is a one-time pass that walks an existing project's PROJECT docs and extracts the stack-level wisdom into a fresh stack module. After bootstrap, ongoing evolution flows through dreams → deltas → this divergence review.

## What auto-applies

Nothing. STACK PRs go through review like any other doc change.

<!-- Source: dreaming_system_v2.md (Section "Core Divergence Review", adapted for stack layer), authored 2026-05-10 -->
