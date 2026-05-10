# Checklist Maintenance

Checklists encode lessons that were paid for in failure. Their value is in being short and trustworthy — every line on a checklist is a line a human reads under time pressure. Bloat kills checklists faster than neglect does.

This doc covers when checks get added, when they get removed, and how the evolution of checklists feeds back into CORE. For the structural rules of authoring a checklist (sections, format, severity tracking, audit-summary matrix), see [`../documentation/checklist-protocol.md`](../documentation/checklist-protocol.md).

## When to add a check

Add a check only when:

- A specific failure has occurred at least once and the check would have caught it. Hypothetical failures do not earn checks.
- The check is verifiable in seconds, not minutes. A check that requires a 20-minute investigation is not a check; it is a procedure that belongs in a different doc.
- The check cannot be enforced by automation. Anything a hook, lint rule, CI job, or test can verify belongs there, not on a human-read checklist.

A new check enters the checklist as part of the dream/delta cycle: a dream surfaces the failure, the delta extraction proposes the check as a PROJECT PR, the engineers approve it, the check ships.

## When to remove a check

Remove a check when any of these holds:

- The check has fired zero times in the last 90 days and the underlying failure mode has not recurred. Stale checks erode trust in the rest of the list.
- The check has been replaced by automation. Once a hook or CI job enforces the rule, the human-read line is dead weight.
- Two checks overlap. Keep the more specific one; remove the redundant one.
- The check tests for a condition that is no longer reachable (the system removed the failure mode entirely — the check is now a fossil).

Removal also flows through the delta cycle, but the threshold is lower than addition: any dream that names the check as redundant or stale earns a removal PR for human review.

## Evolution triggers

A checklist gets a maintenance review when any of these triggers fire:

- **Recurring ERRATA pointing at the same gap.** If `ERR-NNN` and `ERR-MMM` both have root-cause sentences that match a missing check, the next delta extraction adds the check.
- **Dream patterns naming a check as ignored.** If three dreams in a row report skipping the same check because it was unclear what it required, the check needs rewording, not removal.
- **Post-deploy failure where the checklist passed.** A failure that slipped through the checklist means either the check is missing or the existing check's wording does not catch the failure mode. Update the check; do not blame the human.
- **Net-positive infra change rate.** If the rate of new infrastructure resources per session jumps, the Net Infra Footprint Check ([`../documentation/checklist-protocol.md`](../documentation/checklist-protocol.md) §Post-Deploy / End-of-Session Checklist §0) needs sub-bullets covering the new resource types.

## Special status: Net Infra Footprint Check

The Net Infra Footprint Check is the one check that runs at the end of every session including debug-only sessions. It exists because infrastructure drift is the highest-cost silent failure mode the dream system has observed.

This check does not get removed by the 90-day-stale rule. It earns its slot permanently.

When new infrastructure types are added to a project, the check gets a sub-bullet for that type — not a separate check. The check's job is to force the human to enumerate the delta; the sub-bullets are reminders, not the gate.

## Cadence

A full checklist maintenance pass happens at every divergence review (monthly or end-of-project — see [`project-core-divergence.md`](project-core-divergence.md)). The pass:

1. Lists every check across every checklist in the project.
2. For each, records: last fire date, count of fires in the period, count of failures the check caught.
3. Flags candidates for removal (stale + no recurrence) and candidates for promotion to CORE (fired across multiple projects).

Mid-period maintenance is fine for individual additions or removals through the delta cycle; the divergence pass is the systematic sweep.

## Promotion to CORE

A check promotes to upstream CORE when:

- Two or more independent projects added the same check.
- Both projects' delta histories show the check catching real failures, not theoretical ones.
- The check's wording is project-agnostic — passes the "service name swap" test in [`project-core-divergence.md`](project-core-divergence.md).

A promoted check enters `core/documentation/checklist-protocol.md` as part of the standard checklist set, with a CORE-CHANGELOG entry citing the source projects. Forks pull it in on their next CORE update cycle.

## What auto-applies

Nothing. Checklist additions, removals, and rewrites all go through PR review like any other doc change.

<!-- Source: dreaming_system_v2.md (Section "What CORE Is" + checklist references), authored 2026-05-10 -->
