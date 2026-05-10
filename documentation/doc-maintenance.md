# Documentation Maintenance

Rules for when and how to update documentation, how to structure the repo's knowledge layer, how to fight entropy, and how to produce visual artifacts and runbooks.

> See also: [`documentation/writing-style.md`](writing-style.md) for voice and formatting rules.
> See also: [`practices/feature-lifecycle.md`](../practices/feature-lifecycle.md) for the full documentation trail a feature must leave.
> See also: [`practices/session-protocol.md`](../practices/session-protocol.md) for per-session doc obligations.


---

## The Repo Is the System of Record

All critical context lives inside the repo. Not in external wikis, not in chat platforms, not in conversation history. Agents can only reliably access what is in the repo or in explicitly provided tools. Knowledge absent from the repo does not exist for the agent — the same as being unknown to a new hire joining months later.

Consequence: every architectural decision, agreed convention, and hard-won lesson must land as a committed file before the session ends. If it was only said in chat, it is gone.


---

## Documentation Hierarchy

Organize documentation in four layers of increasing detail:

- **Layer 0 — Entry point** (~100–150 lines): session-start reading list, key commandments, current build status, Documentation Map. An index with pointers, not an encyclopedia.
- **Layer 1 — Domain docs**: architecture, conventions, testing, debugging, data model, infrastructure, deployment, security, functionality audit, feature proposals, workflows.
- **Layer 2 — Subdirectory docs**: design docs (with index), execution plans (`active/` and `completed/`), product specs (with index), references, generated docs.
- **Layer 3 — Inline code comments**: implementation-specific context that belongs only in the source file.

An agent working on a typical task reads Layer 0, follows one pointer to the relevant Layer 1 doc, and dives into Layer 2 only when the task demands it. No task requires reading all of Layer 2.

The root-level entry point is a navigation aid, not a content store. Keep it lean. Each doc it points to is self-contained for its scope — an agent reading only that doc must understand its domain without needing the entry point open.

### Recommended top-level directory layout (full-stack mono-repo)

```
<project>/
├── backend/        # API server + business logic
├── frontend/       # Client application
├── infra/          # Infrastructure-as-code templates + deploy scripts
├── docs/           # Architecture, conventions, testing, and design docs
│   ├── design-docs/
│   ├── exec-plans/
│   │   ├── active/
│   │   └── completed/
│   ├── product-specs/
│   ├── references/
│   └── generated/
└── .github/workflows/   # CI/CD pipelines
```

Each area has its own README with local setup instructions.


---

## Documentation Map

Maintain a Documentation Map table in the entry-point file mapping every documentation file to a one-line scope description. This map is the canonical index used during sessions to locate domain-specific guidance without opening every doc.

Recommended categories for any project's Documentation Map:

- Working rules / agent behavior
- Meta-rules / harness guidelines
- Git workflow
- Conventions and code style
- Engineering values / core beliefs
- Dev environment
- Architecture
- Debugging protocol (mark MANDATORY)
- UI review checklist (mark MANDATORY for UI changes)
- Post-deploy / end-of-session checklist (mark MANDATORY)
- Testing strategy
- Infrastructure overview
- Deployment guide
- Data model
- Security
- Feature proposals / backlog
- Execution plans (active / completed)
- Design docs
- Product specs
- Human-facing READMEs

### Entry-point conventions doc summary

The entry-point file carries a brief (4–6 bullet) summary of the project's most important conventions — enough to orient the agent at session start — with a pointer to the canonical conventions doc for full details. Do not duplicate the full conventions doc in the entry point.

### Core beliefs doc update policy

Update the core beliefs doc only when the team's fundamental engineering philosophy changes. Adding a new belief requires explicit discussion. Cap the list at ~15 beliefs — a longer list stops being actionable.


---

## Update Triggers

### After a feature ships to production

Align every documentation artifact with the actual implementation, not the original plan.

- **Execution plan**: If fully complete and verified, move from `active/` to `completed/`. Before archiving, rewrite to reflect actual implementation — replace forecast blocks with "what this plan actually delivered" summaries naming actual files, commits, and test counts. If partially shipped, update in place.
- **Plan file (session record)**: Apply the four-step alignment:
  1. Rename to match the exec-plan / feature name in kebab-case.
  2. Rewrite as an execution record — replace speculation with what was delivered.
  3. Add a "Final state" block: branch, commit hashes, test counts, deploy status, pending items.
  4. Add a "Session reflection" block.
- **Product spec and design doc**: Update acceptance criteria, API contracts, UI behavior, architecture diagrams, data flow, and decision records to match what was built.
- **Functionality audit**: Add new features, move planned items to implemented, resolve gaps the deploy fixes, update summary counts.
- **Infrastructure doc**: Update resource counts, add new resources, update IaC stack descriptions, update cost estimates.
- **Testing doc**: Run the actual test suite — do not guess. Update test counts, add new patterns or fixtures, update coverage numbers.
- **Entry-point file**: Update build status table and milestone section.
- **README files**: Update milestone checklists, test counts, feature lists, infrastructure tables. Verify quick-start commands still work.

### After a bug fix or hotfix

1. Update the entry-point file build-status counts if test counts changed.
2. Add a row to the debugging decision record: Date, Issue, Root Cause, Fix, Lesson.
3. If the bug was a tracked gap, mark it resolved in the functionality tracker.
4. Update the infrastructure doc if the fix involved infrastructure changes.
5. If a plan file exists for this session, apply the full four-step alignment procedure above.

### Quick-reference: change type → docs to update

| Change type | Documentation to update |
|---|---|
| New data store table | Infrastructure doc (count), deployment doc (creation steps), verify table exists in production |
| New API endpoint | Functionality audit, API reference / backend README |
| New frontend page or component | Functionality audit, frontend README |
| New test file | Testing doc (counts), entry-point file (build status), READMEs |
| New IaC resource | Infrastructure doc, deployment doc, verify in production |
| New environment variable | Infrastructure doc, deployment doc |
| Bug fix | Debugging decision record; errata log if severe |
| Milestone completed | Entry-point file (milestones), README, archive exec plan |
| New IAM permission | Infrastructure doc, verify in IAM console |
| New object storage bucket | Infrastructure doc, deployment doc, verify exists |


---

## Update Cadence

Three cadences govern documentation updates:

1. **Per-commit**: Update any doc that tracks implemented functionality, quality grades, or test counts.
2. **Per-milestone**: Run a full audit loop. Update all summary documents. Archive stale documentation. Move completed execution plans from `active/` to `completed/`. Rescan and regrade quality scores.
3. **Per-session (agent)**: Check the machine-specific pitfalls log before running commands. Add new pitfalls after discovering them. Apply the feedback loop protocol after any recurring agent mistake.

> See also: [`practices/git-workflow.md`](../practices/git-workflow.md) for commit and merge procedure.


---

## Functionality Audit and Feature Proposals

### Functionality audit

Maintain a living functionality audit updated after every commit that changes functionality. Structure it as:

1. Implemented features (with backend / frontend / test references)
2. Planned items for upcoming milestones
3. Numbered gaps (sequential, never reused) with severity ratings (High / Med / Low)
4. Suggested features (sequential prefix, never reused)
5. Summary counts by layer and severity

Rules:
- Edit rows in place — never rewrite whole sections.
- Never remove a gap without verifying the fix is committed and tests pass.
- Never add speculative features to Implemented.
- Gap numbers are permanent and must never be reused — downstream references depend on them.

### Feature proposals

Maintain a separate feature-proposals document for non-committed future ideas. Proposals are sequentially numbered (e.g., P1, P2, …) and numbers are never reused. Distinguish small tactical suggestions (tracked in the functionality audit) from larger strategic proposals (tracked here). Record every approve / defer / reject decision in a Decision Log — never delete rows. Proposals move to the functionality audit's Implemented section only after code is committed and tested.


---

## Fighting Entropy

Agents replicate existing patterns — good and bad. Without active maintenance, quality drifts downward. Stale documentation actively misleads; remove outdated content rather than letting it accumulate.

### What decays and why

| Thing | How it decays | Consequence |
|---|---|---|
| Documentation | Becomes stale as code changes | Agent builds on wrong assumptions |
| Test descriptions | Drift from what tests actually verify | False confidence in coverage |
| Naming conventions | Inconsistencies creep in across files | Agent cannot find things by pattern |
| Dead code | Unused functions, orphaned files | Agent copies bad patterns |
| Dependencies | Security vulnerabilities, breaking changes | Build failures, security gaps |
| Quality scores | Grades do not reflect actual state | Wrong prioritization |

### Countermeasures by frequency

| Decay type | Countermeasure | Frequency |
|---|---|---|
| Stale docs | Compare doc claims against actual code | Every milestone |
| Dead code | Run coverage + unused-export analysis | Monthly |
| Naming drift | Scan for convention violations | Per PR |
| Dependency rot | Run security audit tools | Weekly |
| Test drift | Review test descriptions vs. assertions | Every milestone |
| Quality scores | Rescan and regrade per domain | Every milestone |

Technical debt is a high-interest loan — pay it down in small continuous increments, not in painful bursts.

> See also: [`practices/anti-patterns.md`](../practices/anti-patterns.md) for patterns that accelerate decay.


---

## Archive vs. Delete

Archive superseded design and spec documents rather than deleting them. Mark source code as the authoritative implementation reference. When archived specs conflict with the code, the code wins. Documentation describes intent; the running system is the truth.

Update the deployment guide after: adding or removing infrastructure stacks, changing deployment prerequisites, modifying CI/CD pipelines, or updating environment variable definitions. Archive superseded deployment guides at a versioned path rather than deleting them.


---

## README Files

README files are human-facing onboarding docs — targeting new developers, repository visitors, and external collaborators. They are not agent context (that is the entry point plus `docs/`). They are version-controlled and must stay accurate.

Rules:
1. All counts (tests, files, endpoints) must match authoritative sources — run the test suite when in doubt.
2. READMEs summarize; they do not replace detailed architecture or conventions docs — one line per item.
3. Quick-start commands must work on a fresh clone — no machine-specific paths.
4. Feature lists reflect only implemented, tested functionality — never list planned features as present.
5. Each type of information lives in exactly one README — do not repeat the same content across multiple READMEs.

When updating `docs/` during a milestone, always check READMEs for stale counts (test numbers, stack counts, milestones).


---

## Errata Log

The errata log documents catastrophic errors made during agent sessions. Every agent reads this file before performing file operations on sensitive or local-only files.

**Update policy**: Add an entry after any severe mistake that caused data loss, broken state, or required manual recovery. Entries are permanent — never remove them.

### Template for new entries

```
## ERR-NNN: [Short Title]

**Date:** YYYY-MM-DD
**Severity:** CRITICAL | HIGH | MEDIUM
**What happened:** [Factual description of the error]
**Impact:** [What was lost, broken, or degraded]
**Root cause:** [Why the error occurred — the reasoning failure, not just the action]

### Rules to Prevent Recurrence

1. [Specific, actionable rule]
2. [Another rule if needed]
```


---

## Generated Documentation

Some docs must be auto-generated from code to guarantee freshness.

Examples:
- **API catalog**: generated from router / controller files and schema definitions; regenerate after any router change.
- **Component inventory**: generated from the frontend components directory; regenerate after any component change.
- **Data store schema doc**: generated from infrastructure-as-code templates; regenerate after any schema change.

Rules for generated docs:
- Always include a header: `<!-- GENERATED FILE — DO NOT EDIT MANUALLY -->`
- Include the generation command so any agent can regenerate.
- Never mix manual content into generated files.


---

## Screenshot and Visual Artifact Workflow

Screenshots are captured from **mock data pages** — standalone UI pages that render realistic content with hardcoded data, requiring no authentication or live backend. This produces correct screenshots without a populated data store or running services.

### Mock page requirements

Each mock page must be self-contained with:
- A mock navigation bar mirroring the real nav with role-appropriate items
- Hardcoded realistic data (names, scores, dates, sample content)
- Real production UI components using the same styling system and icon library as production
- Responsive layout that renders at both desktop and mobile viewports

### Adding a new mock page

1. Create a new page file under the screenshots route directory.
2. Export a default component with hardcoded mock data.
3. Use the project's mock nav pattern from existing mock pages.
4. Keep styles consistent with production components.
5. Add responsive classes for mobile capture support.

### Capture approach

Start the dev server, then run a headless-browser script that:
1. Opens each mock page at the target viewport.
2. Emulates the desired color scheme.
3. Waits for network idle plus a brief settle delay.
4. Removes any dev-mode overlays injected by the framework.
5. Clips and saves a PNG at the configured output path.

Install the headless browser driver once per machine before running captures.

Key parameters:

| Parameter | Purpose | Desktop | Mobile |
|---|---|---|---|
| `viewport.width` | Screenshot width in pixels | 1280 | 375 |
| `viewport.height` | Browser viewport height | 900 | 812 |
| `clip.height` | Cropped output height (per page) | varies | 812 |
| `colorScheme` | Force light / dark mode | dark | dark |
| `waitUntil` | Page-load signal before capture | networkidle | networkidle |

Each page has a distinct `clip.height` to control how much is captured. Increase to show more content; decrease to crop tighter. Remove framework dev overlays programmatically before the clip.

### Refreshing screenshots after a UI change

1. Update mock page data or layout if the changed UI requires it.
2. Start the dev server.
3. Run the capture script (adjust the port if needed).
4. Verify all images look correct before committing.
5. Commit the updated image files.


---

## Runbook Structure

One-time manual configuration runbooks require a specific structure to prevent future readers from assuming absent features were overlooked rather than deliberately excluded.

### Required sections

1. **What is NOT automated** — state upfront that this is a manual runbook: no IaC, no pipeline, no automation (list the specific scope).
2. **Wiring points** — the number of top-level steps at a glance.
3. **End-to-end flow** — the full redirect or operation chain as a step-by-step sequence so the reader can verify each hop at smoke-test time.
4. **Defense-in-depth note** — never trust a single control; enforce constraints server-side so that a misconfiguration in one layer does not grant access.
5. **What this does NOT do** — explicitly list:
   - Deferred automation and the rationale for each deferral.
   - The compensating control that makes each omission safe.

The "What this does NOT do" section prevents future readers from treating a deliberate exclusion as a gap to fill without understanding the trade-offs.

> See also: [`practices/infra-management.md`](../practices/infra-management.md) for infrastructure change approval rules.
> See also: [`documentation/checklist-protocol.md`](checklist-protocol.md) for checklist structure rules.


---

## Harness Guidelines Provenance

These maintenance patterns synthesize principles from:

- OpenAI: Harness Engineering — original concept and lessons from building a large codebase with AI agents.
- Martin Fowler: Harness Engineering — analysis of the three-layer harness model (context, constraints, correction).
- mtrajan: Harness Engineering Is Not Context Engineering — the critical distinction between giving agents information vs. building systems that prevent, measure, and correct.
- can.ac: The Harness Problem — how the interface layer between agent output and code changes is the highest-leverage place to invest.

---

<!-- Assembled from: architecture.md §Historical Reference, CLAUDE.md §Conventions (summary), CLAUDE.md §Documentation Map, core-beliefs.md §Core Beliefs (preamble / meta-policy), deployment.md §Deployment Guide (preamble / when-to-update), ERRATA.md §Preamble, ERRATA.md §Template for New Entries, harness-guidelines.md §Harness + Context Guidelines (preamble / intro block), harness-guidelines.md §2.1 Repository as System of Record, harness-guidelines.md §2.2 Map Not Manual, harness-guidelines.md §2.3 Progressive Disclosure, harness-guidelines.md §2.7 Fight Entropy Actively, harness-guidelines.md §4.1 Full Document Hierarchy, harness-guidelines.md §4.5 README Files Role, harness-guidelines.md §6.1 What Decays, harness-guidelines.md §6.2 Countermeasures, harness-guidelines.md §8 Generated Documentation, harness-guidelines.md §Sources, post-deploy-checklist.md §3 Documentation Updates — Feature Deploys, post-deploy-checklist.md §4 Documentation Updates — Bug Fixes / Hotfixes, post-deploy-checklist.md §Quick Reference What to Update When, workflows.md §Functionality Doc Maintenance, workflows.md §Feature Proposals Doc, workflows.md §Documentation Maintenance Cadence, workflows.md §README Maintenance, README.md §Project Structure, sso-idp-setup.md §intro / flow diagram, sso-idp-setup.md §What this does NOT do, screenshots.md §Overview, screenshots.md §Mock Pages, screenshots.md §Capturing Screenshots, screenshots.md §Updating Screenshots -->
