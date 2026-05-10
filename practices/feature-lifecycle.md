# Feature Lifecycle

Every feature follows a documented trail from proposal to archival. Skip any step and the artifact chain breaks — future sessions cannot reconstruct why decisions were made or what was actually built.

> See also: [plan-mode.md](plan-mode.md) for design-approval gates, [infra-management.md](infra-management.md) for infrastructure change sequencing, [git-workflow.md](git-workflow.md) for commit and merge discipline, [session-protocol.md](session-protocol.md) for session-start and session-end requirements.

---

## Documentation Trail

Every feature must produce this chain of artifacts before its execution plan is archived:

```
product_spec → design_doc → exec_plan (active)
  → [DEVELOPMENT]
    → User verifies completion + tests pass
      → UPDATE product_spec, design_doc, exec_plan to reflect what was built
        → UPDATE all other affected docs
          → ONLY THEN move exec_plan to completed/
            → Reflect: do working-rules docs or the error log need updates?
```

The execution plan must not move to completed until:

1. All acceptance criteria in the execution plan are met.
2. All tests pass at every layer.
3. Documentation is updated per the maintenance cadence.
4. The spec, design doc, and execution plan reflect what was actually built — not the original proposal.
5. The user has verified the feature works.
6. All other affected docs are updated.

A feature is not complete until docs reflect reality. Docs update before the plan archives — not after.

> See also: [documentation/checklist-protocol.md](../documentation/checklist-protocol.md) for the post-commit doc update checklist, [documentation/doc-maintenance.md](../documentation/doc-maintenance.md) for maintenance cadence.

---

## Before Writing Code

Never start implementation of a non-trivial feature without first creating: a product specification, a design document (if new infrastructure is involved), and an execution plan in the active work queue.

Self-check before the first commit: "Have I created the spec, design doc, and exec plan?" If not — stop and create them.

Design approval does not authorize skipping the documentation trail. Design approval captures the approach; the documentation trail formalizes it as versioned artifacts that persist across sessions.

After context compaction or session resumption: re-read project behavioral rules before resuming work. A resumption summary is not a substitute for the behavioral contract.

> See also: [practices/session-protocol.md](session-protocol.md) for session-start protocol.

---

## When to Use the Full Process

| Trigger | Required phases |
|---|---|
| New data store | All phases |
| New subsystem | All phases |
| Cross-cutting feature (affects 3+ existing services) | All phases |
| Single feature in an existing subsystem | Proposals + Spec + Implementation (skip architecture if no new infrastructure) |
| Bug fix or small enhancement | None — implement and update docs |

---

## Phase Sequence

Major changes follow a structured sequence. Each phase produces a versioned artifact before the next phase begins. This prevents architecture decisions from being made in a vacuum and keeps documentation synchronized with design.

**Phase 1 — Requirements Gathering**
- Read existing proposals, audit docs, and business-model docs.
- Identify requirements with source references.
- Flag gaps: requirements not covered by existing proposals.
- Output: requirements list.

**Phase 2 — Scoping and Formal Proposals**
- Scope each requirement (include / defer / reject).
- Write formal feature proposals in the proposals doc.
- Update the cross-reference table; supersede old proposals if applicable.
- Output: proposals doc updated.

**Phase 3 — Architecture Decision**
- Analyze at least two options against core beliefs, existing stack, cost, and complexity.
- Present trade-offs with concrete numbers (cost, latency, migration effort).
- User selects the approach.
- Write a numbered design decision doc.
- Cascade updates to all affected docs (data model, infrastructure, architecture, etc.).
- Output: design decision doc created, affected docs updated.

**Phase 4 — Product Spec**
- Write a numbered product spec covering user stories, data model, API surface, and known gaps.
- Reference the design decision and proposals from prior phases.
- Update the product spec index.
- Output: product spec created.

**Phase 5 — Execution Plan**
- Break implementation into ordered phases with dependencies.
- Each phase: scope, files to create or modify, tests, acceptance criteria.
- Place the plan in the active plans directory.
- Reference the design decision and product spec.
- Output: execution plan in the active plans directory.

**Phase 6 — Implementation**
- Read the full context chain before writing code.
- Follow the execution plan phase-by-phase.
- Update execution plan progress after each phase.
- Run the post-commit doc update checklist on every commit.
- When complete: verify all six gates (see Documentation Trail above), then move the execution plan to the completed archive.

---

## Artifact Dependency Graph

```
Requirements
  └─→ Feature Proposals
        └─→ Design Decision
              ├─→ Product Spec
              └─→ Cascading doc updates (data model, infrastructure, architecture, etc.)
                    └─→ Execution Plan
                          └─→ Implementation
```

Each artifact references its upstream artifacts by ID. This creates a traceable chain from business requirement to committed code.

---

## Execution Plans as First-Class Artifacts

Plans are versioned, co-located with the codebase, and independently loadable. Active plans live in a dedicated active directory. Completed plans move to a completed directory. Each plan carries its own progress state so work can resume without reconstructing context. An agent working on feature X loads only plan X — plans do not crowd out other context.

---

## Feature Proposals

The proposals doc tracks larger feature ideas that might be built. These are research-stage candidates, not commitments.

A separate audit doc tracks what is built, what is broken (gaps), and small tactical suggestions. The proposals doc tracks candidates for evaluation and prioritization.

Proposals stay in the proposals doc until a decision is made. When approved: create a milestone entry and track in the audit doc. When rejected: add a one-line rationale in the Status column and leave the row — never delete, for audit trail.

### Proposal Lifecycle

```
PROPOSED → EVALUATING → APPROVED (moves to milestone plan)
                      → DEFERRED (keep, revisit later)
                      → REJECTED (keep with rationale)
```

### Numbering

Proposals are numbered sequentially (`P1, P2, ...`). Numbers are never reused and never renumbered.

### Priority Tiers

- **Critical** — Competitive table stakes. Will lose deals without this.
- **High** — Strong growth driver. Needed within the first six months post-launch.
- **Medium** — Meaningful improvement. Plan for when resources allow.
- **Low** — Nice-to-have or speculative. Build only if effort is trivial.

### Cross-Reference to the Audit Doc

When a proposal is approved:
- Identify any overlapping gaps or suggestions in the audit doc.
- Either remove the audit item (if the proposal fully supersedes it) or resolve it (if it covers a gap).
- Track via milestone going forward.
- Maintain a cross-reference table mapping proposal numbers to overlapping audit items and the required consolidation action.

### Decision Log

Record decisions as proposals are evaluated. One line per decision. The log is append-only — never delete rows.

| Date | Proposal | Decision | Rationale |
|---|---|---|---|

Decisions to record: PROPOSED, EVALUATING, APPROVED, DEFERRED, REJECTED, IMPLEMENTED, SUPERSEDED, ARCHITECTURE DECIDED, SPEC + PLAN CREATED.

---

## Shipping Bias

Bias toward shipping and fixing over blocking and debating. Corrections are cheap; waiting is expensive. Never compromise on data integrity or security — those corrections are not cheap.

> See also: [practices/anti-patterns.md](anti-patterns.md), [practices/security-principles.md](security-principles.md).

---

<!-- Assembled from: agent-behavior.md §9. Feature Development Lifecycle, CLAUDE.md §Three Commandments / 3. Documentation trail for features, core-beliefs.md §Process — belief 14 (Corrections are cheap waiting is expensive), ERRATA.md §ERR-002: Feature Implemented Without Documentation Trail, feature-proposals.md §Feature Proposals (header + preamble), feature-proposals.md §How This Document Works, feature-proposals.md §Cross-Reference to functionality-audit.md, feature-proposals.md §Decision Log, harness-guidelines.md §2.8 Plans as First-Class Artifacts, harness-guidelines.md §3. Major Change Process (section intro), harness-guidelines.md §3.1 Phase Sequence, harness-guidelines.md §3.2 When to Use This Process, harness-guidelines.md §3.3 Artifact Dependency Graph, workflows.md §Verification Gate — Before Marking Work Complete -->
