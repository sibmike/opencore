# Checklist Protocol

Checklists and audit artifacts encode verified lessons. This doc covers: when to run each class of checklist, how to author and update them, the living-audit methodology, quality-tracking artifacts, and the suggestion-backlog discipline.

> See also: [documentation/doc-maintenance.md](../documentation/doc-maintenance.md) for freshness rules; [practices/feature-lifecycle.md](../practices/feature-lifecycle.md) for milestone gates; [practices/debugging.md](../practices/debugging.md) for the debugging protocol.

---

## Post-Deploy / End-of-Session Checklist

### When to run

Run after any of these events — no shortcuts:

| Event | Required sections |
|-------|-------------------|
| Full feature deploy | All sections |
| Hotfix / targeted bug-fix deploy | Infrastructure verification + smoke test + doc updates |
| Debugging session with no deploy | Doc updates + error log + session reflection |
| Infrastructure-only change | Infrastructure verification + doc updates |

At minimum one section is mandatory for every session, including debug-only sessions. This catches silent drift before it ships.

### Contents

Run these checks in order:

**0. Net infra footprint check.** Count and document the session's infrastructure delta in the commit message and execution plan. If non-zero and plan-mode approval was skipped, flag it as a process violation. Even when delta is zero, record it explicitly.

**1. Infrastructure verification.** Confirm all resources exist in the deployment environment. Local tests mock external services — they do not prove resources exist in production. Run the appropriate verification commands against the live environment.

**2. Production smoke test.**
1. Clear browser cache and service worker state; hard refresh.
2. Verify the health/liveness endpoint returns healthy.
3. Complete the full authentication flow with a test account.
4. Navigate to each major page; verify no runtime errors.
5. Inspect network requests in DevTools — verify all API calls return success (not 5xx, not CORS errors).
6. Test with accounts in each user role — different roles hit different code paths.

When an automated end-to-end suite exists, run it against the production environment after deploy. Store results in a central artifact location with a date-stamped path for audit trail.

**3. Documentation updates.** Align all affected docs — execution plans, specs, design docs, feature audit, infrastructure docs, testing docs, root config. Docs must reflect the current system state before the session closes.

**4. Error log.** Add an entry if severe mistakes were made (three or more failed fix iterations, misdiagnosis, or broken production).

**5. Machine/environment pitfalls log.** Add any new environment-specific pitfalls discovered.

**6. Session reflection.** Was the debugging protocol followed? Were behavioral rules obeyed?

### Updating this checklist

Update after a post-deploy failure reveals a missing check, after a new infrastructure pattern is introduced, or when a new deployment target is added.

> See also: [practices/session-protocol.md](../practices/session-protocol.md) for session-start rules; [practices/infra-management.md](../practices/infra-management.md) for plan-mode approval requirements.

---

## Deployment Debugging Checklist

When a deployed app does not work but local dev does, check these categories in order.

**1. Is the app reachable?**
- DNS resolves correctly (domain points to the hosting service).
- HTTPS certificate is valid and active.
- Hosting service is healthy and not misconfigured.
- The correct deployment is live (build logs show the expected commit hash).

**2. Does the build succeed?**
- Build logs show success, not failure.
- The correct branch is deployed.
- Build output matches the hosting service's expected artifact paths.

**3. Are environment variables reaching the app?**
- Variables are set in the hosting console.
- Variables available at build time — client-side variables get inlined at build, not runtime.
- Variables available at runtime for server-side code paths.

**4. Is the frontend talking to the backend?**
- API URL is correct (check actual network request URLs in DevTools).
- CORS is configured — the backend allows requests from the frontend's origin.
- Requests are reaching the backend (check backend logs, not just frontend errors).
- Auth tokens are attached to requests (check the Authorization header).

> See also: [practices/debugging.md](../practices/debugging.md) for the full Observe → Hypothesize → Verify → Plan → Implement protocol.

---

## UI Review Checklist

### Rule

You are not done until you have looked at every affected section at both a narrow mobile viewport and a standard desktop viewport. "I think it's fine" is not verification. "I checked the sections I changed" is not verification. Verification means: take screenshots, look at each one, document any issues, fix them, then re-verify.

### When to apply

Before committing any change that modifies visual text, headings, card layouts, or responsive structure on user-facing pages. Shotgun fixes without systematic verification produce orphans, mid-word hyphenation, forced line breaks, and horizontal overflows that look fine on one viewport but break another.

### Verification workflow

1. Start the dev server; open the app in your browser or preview tool.
2. Audit at desktop width (e.g. 1280 px): navigate to each affected page; for scrollable pages scroll through every section and screenshot it; for paginated views iterate through every slide or page.
3. Audit at mobile width (e.g. 375 px): same process — every section, every slide, screenshot each one.
4. For each screenshot, check:
   - No orphaned words (single word alone on last line).
   - No mid-word hyphenation.
   - No mid-phrase breaks that separate logically-bound words.
   - No text clipped at the right edge.
   - No text too small to read at minimum readable size.
   - Grids stack properly on mobile (no three or more columns at narrow viewport).
   - Fixed-width elements fit within the viewport.
   - Images and screenshots are visible and readable at rendered size.

### Viewport-specific checks

**Mobile viewport:**
- Is the navigation still usable? Are items appropriately hidden or collapsed?
- Are headings readable without mid-word breaks?
- Do multi-column grids stack vertically?
- Are fixed-width elements constrained to fit within the viewport?
- Do absolute-positioned decorations cause horizontal overflow?

**Desktop viewport:**
- Do line breaks added for mobile cause awkward forced breaks on desktop?
- Do non-breaking-space clusters force unnatural spacing or over-long unbroken phrases?
- Do headings that looked fine on mobile now look too small on desktop?
- Are multi-column layouts using the available width?
- Do images render at their full intended resolution?

### Pre-commit script

Run this for any commit touching user-facing visual layout:

1. Dev server running? Start it.
2. Desktop audit: navigate and screenshot each affected section at desktop width.
3. Mobile audit: navigate and screenshot each affected section at mobile width.
4. Issue inventory: write down every bad break, clip, or orphan. Collect first — do not fix yet.
5. Apply fixes: one fix at a time when possible.
6. Re-verify: screenshot the affected sections again on both viewports.
7. Static type / lint check: run your project's type checker or linter.
8. Commit: only after all above steps pass.

---

## Agent-Assisted Constraints

Apply these in any AI-assisted development workflow:

- **Doc freshness check (before commit):** Flag when changed files have no corresponding doc update.
- **Commit message review (before commit):** Verify imperative mood and concise summary.
- **Architectural conformance (during code review):** Validate new code follows established patterns.
- **Test coverage for new code (after implementation):** Ensure new features have tests.

> See also: [practices/git-workflow.md](../practices/git-workflow.md) for commit discipline; [practices/testing-principles.md](../practices/testing-principles.md) for coverage targets.

---

## Post-Milestone Audit Loop

After every milestone completion:

1. Scan all docs against current code state.
2. List discrepancies (outdated counts, missing features, wrong file paths).
3. Update the quality score doc with current grades.
4. Human reviews and approves fixes.
5. Apply fixes and update freshness markers.
6. Open targeted refactoring PRs for any code issues found.

> See also: [practices/feature-lifecycle.md](../practices/feature-lifecycle.md) for the full milestone gate sequence.

---

## Living Audit Methodology

A functionality audit is a living document that tracks implemented, planned, and missing features across all layers. It must reference an update-rules section in the team's workflow documentation to clarify when and how entries are added or resolved.

### Sections and entry structure

**Implemented.** A by-domain layer table: counts of implemented, planned, missing, and suggested items per layer.

**Planned but not yet implemented.** Features explicitly documented in specs and scheduled for a future milestone. Each entry: current state, target behavior, spec reference. This makes scope boundaries auditable and prevents double-counting with gaps.

**Missing functionality (gaps).** Features specified in documentation but absent from code, beyond current planned-milestone scope. Each gap entry:
- Stable numeric ID.
- Description.
- Spec reference.
- Impact statement.
- Severity rating: High / Medium / Low.

Separate gaps into domain sub-sections (Auth, UI/UX, Backend, Infrastructure, Testing) for navigability.

**Suggested functionality.** Features not present in any spec but warranted by product trajectory. Each suggestion:
- Stable S-number.
- Brief rationale tied to product behavior (not vague "nice-to-have" language).
- Rough effort estimate: Low / Medium / High.

Keep suggestions clearly separated from numbered gaps. Suggestions are proactive discovery; gaps are spec-vs-code deficits.

### Strike-through discipline

Strike through resolved items in-place rather than deleting them. This preserves audit history, makes closure visible, and keeps the original gap ID stable for reference in commit messages and execution plans.

### Summary matrix

A functionality audit summary matrix provides at minimum:
1. A by-domain layer table showing counts of implemented, planned, missing, and suggested items per layer.
2. A by-severity rollup of all open gaps.
3. A "Pre-Launch Blockers" list of gaps that must close before production go-live.
4. A "Post-Launch Priority Queue" ordering remaining work.

Resolved items in all lists are struck through in-place to preserve audit lineage.

---

## Quality-Tracking Artifacts

### Grading scale

| Grade | Meaning | Criteria |
|-------|---------|----------|
| **A** | Production-ready | Feature complete, tested, no known gaps |
| **B** | Functional with minor gaps | Core features work, minor edge cases or polish missing |
| **C** | Functional with significant gaps | Works for happy path, notable missing pieces |
| **D** | Partially implemented | Stubs or incomplete, not production-ready |
| **F** | Not started or broken | Missing entirely or non-functional |

### Update cadence

Update quality grades after every milestone completion, after resolving a gap from the functionality audit, or after discovering a new quality issue during development.

### Summary roll-up table

Maintain a roll-up table aggregating grade counts per layer across all grade levels:

| Layer | A | B | C | D | F |
|-------|---|---|---|---|---|
| Backend | — | — | — | — | — |
| Frontend | — | — | — | — | — |
| Infrastructure | — | — | — | — | — |
| **Total** | — | — | — | — | — |

### Pre-launch blockers

Any domain below grade B is a pre-launch blocker. Track blocked domains here. Strike through items when resolved, noting what changed and what grade was achieved.

### Post-launch priority queue

Maintain an ordered post-launch improvement queue. Strike through items when resolved, noting what changed. Items typically include: automated infrastructure tests, CI coverage gaps, missing cross-cutting concerns.

---

<!-- Assembled from:
  agent-behavior.md §11. Post-Deploy / End-of-Session Checklist,
  debugging.md §3. Deployment Debugging Checklist,
  functionality-audit.md §Preamble,
  functionality-audit.md §2. Planned but Not Yet Implemented,
  functionality-audit.md §3. Missing Functionality,
  functionality-audit.md §4. Suggested Functionality,
  functionality-audit.md §5. Summary Matrix,
  harness-guidelines.md §5.4 Agent-Assisted Constraints,
  harness-guidelines.md §6.3 The Audit Loop,
  post-deploy-checklist.md §Preamble,
  post-deploy-checklist.md §When to Use This Checklist,
  post-deploy-checklist.md §2. Production Smoke Test,
  quality-score.md §Preamble,
  quality-score.md §Grading Scale,
  quality-score.md §Summary,
  quality-score.md §Pre-Launch Blockers,
  quality-score.md §Priority Queue,
  ui-review-checklist.md §Preamble,
  ui-review-checklist.md §The Cardinal Rule,
  ui-review-checklist.md §1. Verification Workflow,
  ui-review-checklist.md §4. Viewport-Specific Checks,
  ui-review-checklist.md §5. Pre-Commit Verification Script,
  workflows.md §UI Review Checklist,
  workflows.md §Post-Deploy Checklist
-->
