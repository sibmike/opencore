# Delta Extraction

Delta extraction batch-processes dreams into actionable PRs. Individual dreams are noisy — convergence across dreams is signal. The extraction pass aggregates, filters, and produces grouped PRs the project engineers and human can review.

## When to run

Run an extraction every five new dreams, or weekly, whichever comes first. Five is the floor — extracting too early surfaces noise as if it were signal. Two-week gaps are tolerable; longer than that and dreams pile up faster than they get reviewed.

## Output

Write the batch to `<project>/docs/deltas/delta-batch-NNN.md`, where `NNN` is the next zero-padded integer (`001`, `002`, ...). Use the template at [`../templates/delta-report.md.template`](../templates/delta-report.md.template).

## Read first (mandatory inputs)

- The dream batch in `<project>/docs/dreams/`. Specify exactly which dreams the batch covers (filename range).
- Previous deltas in `<project>/docs/deltas/`. Avoid re-proposing items the prior batch already rejected, deferred, or deemed contested.

## The prompt (run verbatim)

<!-- Source: dreaming_system_v2.md lines 314-350 -->

```markdown
# Delta Extraction: Batch Processing

Process a batch of dream reports to produce actionable PRs.

## Inputs
- Dream batch: <project>/docs/dreams/ (specify which ones)
- Previous deltas: <project>/docs/deltas/ (avoid re-proposing rejected items)

## Process

### Aggregate
Read all dreams in batch. Build frequency map:
- Recurring deltas (convergence = high signal)
- Recurring core-practice drifts (strongest candidates for eventual CORE update)
- Recurring user patterns

### Filter
- Proposed 2+ times independently → strong candidate
- Proposed once, high-confidence → candidate
- Proposed once, low-confidence → hold for next batch
- Contradictory proposals → preserve both, flag as contested

### Produce

Write to: `<project>/docs/deltas/delta-batch-NNN.md`

Contents:
1. PROJECT PR bundle (ready for engineer review)
2. Core-practice drift summary (what forked files changed and why)
3. USER PR bundle (ready for human review)
4. ERRATA entries (ready to apply)
5. Contested items (need human judgment)
6. Deferred items (insufficient evidence, revisit next batch)
```

## Filter thresholds (concrete)

The Filter step's "high-confidence" and "low-confidence" labels are judgment calls. Apply these rules to keep them honest:

- **Proposed 2+ times independently → strong candidate.** Ship in the PROJECT PR bundle. The two dreams must be from independent sessions — sequential dreams that quote each other do not count as independent.
- **Proposed once, with concrete diff and matching ERRATA → candidate.** Ship if the ERRATA's root-cause sentence aligns with the dream's reasoning.
- **Proposed once, no ERRATA backing → hold.** Carry forward to the next batch's Inputs section. Three batches without convergence → drop.
- **Contradictory proposals → contested.** Two dreams reach opposite conclusions. Preserve both, surface in the Contested section, do not pick a winner. Human decides.

## Grouping rules

When multiple PROJECT PRs target the same file, bundle them into a single PR with a combined diff. When multiple core-practice drifts touch the same forked file, propose a single coordinated edit so the reviewer sees the full picture.

USER PRs always go into a single bundle — the human reviews the user-preferences file as a whole, not line by line.

ERRATA entries get their own section with the next available `ERR-NNN` IDs assigned. Do not skip numbers; do not reuse retired numbers.

## What an extraction does NOT do

- It does not promote drift to upstream CORE. That is the job of [`project-core-divergence.md`](project-core-divergence.md), which runs on a slower cadence.
- It does not apply any PR. Every output is staged for human review.
- It does not edit the source dreams. Dreams are append-only.
- It does not create new ERRATA entries that the dreams themselves did not propose. Extraction aggregates; it does not invent.

<!-- Source: dreaming_system_v2.md (Section "Delta Extraction Prompt"), authored 2026-05-10 -->
