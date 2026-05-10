# Debugging Protocol

> Systematic methodology for diagnosing and fixing bugs. Debugging is feature engineering — it requires the same rigor as building new features: research, hypothesize, verify, then act.

**When to update this file:** After a debugging session reveals a new anti-pattern, after a misdiagnosis causes a regression, or when a new environment is added.

---

## Core Principle

**Debugging is not trial-and-error.** It is a systematic process of narrowing the problem space until the root cause is identified with certainty, then applying the minimum change to fix it.

The cardinal sin: making changes to a live system without understanding the system. Every "quick fix" that isn't grounded in verified understanding risks creating a new, worse problem.

Trial-and-error pushed to production is not debugging; it is gambling with a working system. Fix one thing at a time — isolate problems, never combine fixes for unrelated issues in one change.

> See also: [practices/anti-patterns.md](anti-patterns.md)

---

## The Five-Phase Protocol

Follow the phases in order. Never skip to implementation.

### Phase 1: Observe — Gather Evidence (no code changes)

1. **Reproduce the exact error.** Get the exact error message, stack trace, HTTP status, or console output.
2. **Identify the boundary.** Where does the error originate? Frontend? Backend? Infrastructure? Network?
3. **Read the logs.** All of them. Backend logs, frontend console, build logs, deployment logs, browser network tab.
4. **Check what changed.** `git log` — what was the last working commit? What changed since then?
5. **Verify the environment.** Are environment variables set? Are services running? Is DNS pointing to the right place? Is the build using the right commit?

**Output of Phase 1:** A written statement: "The error is [X], occurring at [layer], caused by [evidence suggests Y]."

**Critical:** Always check backend logs first when an API call fails. Browser console errors (such as network errors) are symptoms — the backend log shows the cause. The single question "what does the backend log say?" prevents entire classes of misdiagnosis.

Network errors on non-preflight requests are often symptoms of unhandled 500s. When a backend exception bypasses middleware, the browser reports a network or cross-origin error instead of a 500. Always check the backend response status before assuming the problem is in the browser layer.

### Phase 2: Hypothesize — Form a Theory (no code changes)

1. **List all possible causes.** Don't stop at the first plausible explanation.
2. **Rank by likelihood.** Use evidence from Phase 1 to rank — not gut feeling.
3. **Identify what would confirm or refute each hypothesis.** What log line, environment variable value, or network request would prove it?
4. **Design a verification test for the top hypothesis.** A read-only check — not a code change.

**Output of Phase 2:** "My top hypothesis is [X]. I can verify it by checking [Y]. If confirmed, the fix is [Z]."

### Phase 3: Verify — Confirm the Hypothesis (no code changes)

1. **Run the verification test.** Check the environment variable, read the log, inspect the network request, query the service.
2. **If confirmed:** Proceed to Phase 4.
3. **If refuted:** Return to Phase 2 with the next hypothesis. Do not start making changes.

**Output of Phase 3:** "Hypothesis confirmed/refuted. Evidence: [specific data point]."

If the fix doesn't work, stop and re-observe rather than iterating on the same hypothesis. Re-examine the patient before changing the medicine.

### Phase 4: Plan the Fix (no code changes yet)

1. **Identify the minimum change** that fixes the root cause.
2. **Assess blast radius.** What else does this change affect? Other environments? Other features? Build output?
3. **Check for side effects.** Read the code that depends on what you're changing.
4. **Decide: code change vs. config change vs. infrastructure change.** If code works locally but not in deployment, the problem is almost certainly configuration, not code. Don't change code when the problem is config.
5. **If the fix touches deployment or infrastructure:** Verify in a non-production context first if possible.

**Output of Phase 4:** "The fix is [specific change] to [specific file/config]. Blast radius: [what else is affected]. Verification: [how to confirm it worked]."

> See also: [practices/infra-management.md](infra-management.md)

### Phase 5: Implement and Verify

1. **Make the single, planned change.** Nothing extra. No "while I'm here" improvements.
2. **Run tests locally.** All of them.
3. **Verify the fix addresses the original error.** Not "it compiles" — the original reproduction case must pass.
4. **Commit with a clear message** explaining the root cause and why this change fixes it.

> See also: [practices/testing-principles.md](testing-principles.md)

---

## Verification Discipline

Never mark a task complete without proving it works. Run tests, check logs, demonstrate correctness. Ask: "Would a staff engineer approve this?"

Run the actual test suite before updating test counts in docs. Counts drift between sessions.

### Post-Deploy Verification

After every production deployment, verify each integration independently before declaring the deploy complete:

1. Health endpoint — confirms the service is up and environment is loaded correctly.
2. Frontend load — confirms build artifacts and API URL wiring are correct.
3. Authentication flow — confirms the auth service, session handling, and user creation.
4. File upload — confirms storage permissions and cross-origin configuration are correct.
5. Storage security — confirm private objects return Access Denied when accessed directly.
6. Async pipeline trigger — confirm background jobs are enqueued and visible in the orchestration service.
7. Transactional email — confirm the email service is delivering.

Do not skip the security check (item 5). Misconfigured storage permissions are a common silent failure that passes all functional tests.

After every deployment, run a post-deploy smoke test for any feature with role-gated entry points: sign in as the most complex role combination the feature targets, verify every entry point is visible, and confirm the destination renders correctly.

For OAuth integrations specifically:

1. Open a private window to the protected resource — forces a clean auth state.
2. Trigger the SSO entry point and confirm the expected identity-provider UI appears.
3. Authenticate with a valid identity and verify the callback landing page.
4. Confirm the protected resource renders correctly post-redirect.
5. Sign out and verify the post-logout redirect target.

Diagnostic: if sign-in fails with a redirect mismatch error, verify that the callback URL registered with the identity provider exactly matches the app-client configuration — check trailing slashes, scheme, and port numbers. Both sides must be character-for-character identical.

> See also: [documentation/checklist-protocol.md](../documentation/checklist-protocol.md)

---

## Debugging Specific Bug Classes

### Deployment and Configuration Failures

Common deployment pitfalls worth encoding as institutional knowledge:

**Container / managed-runtime platforms:**
- Instance roles must have the correct trust principal for the runtime service. Missing a required subdomain prefix prevents the role from appearing in the console.
- Python 3.11 "revised build" environments discard packages installed in the build phase. Use pre-run hooks for dependency installs; use explicit interpreter aliases rather than bare command names.
- VPC connectors route all outbound traffic through the private network. Without a NAT gateway, public endpoint access from private subnets fails silently — all authenticated requests return 401 because the identity-provider key endpoint is unreachable.

**Static-site / SSR hosting:**
- Framework detection is permanent per branch; fix misdetection via CLI, not the console.
- TypeScript config files may be invisible to the hosting provider's detector; use a JavaScript equivalent.
- Platform console environment variables override committed config files — use the committed file as the single source of truth and remove conflicting platform-level variables.
- SSR monorepo builds require an explicit application root declaration in the build config.

**Infrastructure as code:**
- Template changes to managed data stores (such as new indexes) are not applied automatically. The stack must be updated explicitly after adding indexes.
- Verify SSL/TLS certificate region requirements before requesting certificates; some CDN services require certificates in a specific global region.

**Auth / startup sequencing:**
- Redirect loops between the login page and the dashboard can be caused by a valid session cookie pointing to a dead or degraded backend. Sign out the session on persistent 401 errors.
- Data store connection failures at startup must not crash the application. Use a fallback mode to keep the service alive and preserve debugging access.

### UI and Visual Bugs

Follow the same five-phase protocol for UI bugs. Shotgun-clicking layout adjustments until something looks right is the UI equivalent of trial-and-error production changes.

Before any commit touching user-facing visual code, run a complete audit at all relevant viewport sizes — not just the sections changed, but every section.

When fixing a class of bug (orphans, overflow, layout breaks), search the entire file for the pattern and fix all occurrences in one pass. Don't wait for the user to find each instance.

Systematically verify every section at both narrow and wide viewports before claiming done. A fix that resolves the issue at one viewport but breaks another is not a fix — it is a new bug.

Trust the visual screenshot over DOM measurements. If a screenshot shows clipping, treat it as a real bug worth investigating before dismissing.

### Analytics and Instrumentation Bugs

Every new access path to a resource must be instrumented in the same change that ships the access path. The question "if I add a new way for someone to access X, did I also update the analytics write path?" must be answered explicitly.

Frontend tests must assert the on-the-wire request payload, not just that a tracking call fired. A test that asserts the call was invoked is useless when the bug is that the payload is empty.

Smoke-test user-facing metrics post-deploy with a real two-browser flow: trigger the tracked action with one session, confirm the metric shows non-zero from the observing session.

Don't conflate "I don't want to inflate metric X" with "I shouldn't record the data." Track everything with sufficient context (access method, user role, etc.); apply filtering on the read side. Removing data at write time makes retrospective segmentation impossible.

Duration metrics require event-driven boundaries (visibility, focus, input, navigation) — not polling. A polling accumulator will under-count for events shorter than the interval. Active time is not raw session duration — an idle open tab is not engagement time.

When splitting an aggregate metric across multiple write paths, the read path must reflect all write paths. Aggregate metrics and detail metrics on the same page must use consistent time windows.

---

## Feature Completeness and Verification

A feature is not complete until the user-visible promise is honored end-to-end. "Backend write exists" is a checkpoint, not a finish line.

Every write endpoint that promises to notify a third party must have a test that asserts the notification actually fires — not just that the data store write succeeded. A test that only checks the stored record is incomplete.

Every mutation handler in the frontend must surface both success and error to the user. Silent success equals silent failure. If the action might fail in a way the user cares about, the absence of an error surface is itself a bug.

Any feature with a "user X will see Y" or "user X will be notified about Y" promise must include a test that exercises Y end-to-end from at least two entry points — the in-app flow and any out-of-app entry point such as an email link, deep link, or scan code.

Every data store resource introduced by a feature must be added to the test fixture setup in the same change. Otherwise any future test exercising that resource either fails with a resource-not-found error or is silently skipped.

Pagination defaults on detail-page lookups are a routing trap. When a per-entity detail route is sourced from a paginated list query without an entity-specific lookup, the route silently 404s for entities past page one. Either add a dedicated entity-by-ID endpoint or fetch the specific entity by ID on the detail page — never by "find in page-one list result."

Navigation counts and badges must point at a UI surface that lets the user act on the underlying state. A count with no corresponding action surface is worse than no count.

Every data-fetching consumer must differentiate loading, error, and empty states explicitly. Never collapse these three states into a single loading branch:

- Loading: skeleton or spinner
- Error: error state with retry or contact CTA
- Empty (after successful load): empty state with actionable next step

Every page that depends on a profile or record existing must have a "no record yet" empty state with an actionable next step, not a perpetual skeleton.

> See also: [practices/feature-lifecycle.md](feature-lifecycle.md)

---

## Autonomous Bug Fixing

When given a bug report, fix it. Don't ask for hand-holding. Point at logs, errors, failing tests — then resolve them. Zero context switching required from the user. Go fix failing tests without being told how.

After deploying new features, verify that all new resources exist in the target environment. Local tests mock external services — they prove code logic, not infrastructure existence.

> See also: [practices/session-protocol.md](session-protocol.md)

<!-- Assembled from: agent-behavior.md §6 Autonomous Bug Fixing, debugging.md §Debugging Protocol (preamble), debugging.md §1 Core Principle, debugging.md §2 The Protocol, deployment.md §14 Troubleshooting, ERRATA.md §ERR-003 Misdiagnosed Root Cause, ERRATA.md §ERR-006 Shotgun Debugging of Mobile/Desktop UI, harness-guidelines.md §2.11 Debug Like an Engineer Not a Gambler, workflows.md §Debugging Protocol, agent-behavior.md §4 Verification Before Done, deployment.md §12 Verification Checklist, ERRATA.md §ERR-009 Half-Built Feature Shipped Without Email Path or Toast, ERRATA.md §ERR-010 M16 Request Access Shipped With Persistence + Sidebar Badge But No Working Founder Affordance, ERRATA.md §ERR-011 Analytics Shipped Without Authenticated-Path Instrumentation, ERRATA.md §ERR-012 PS-30 Share QR Buried + No-Profile Page Stuck on Skeleton, google-sso-cognito-setup.md §Step 6 Smoke test -->
