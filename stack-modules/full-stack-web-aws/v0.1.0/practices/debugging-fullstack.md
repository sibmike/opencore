# Full-Stack Debugging: AWS + FastAPI + Next.js

> This doc covers stack-specific failure modes. The generic Observe → Hypothesize → Verify → Plan → Implement protocol lives in `core/practices/debugging.md` — follow it. Come here for the taxonomy of where things go wrong in this stack and how to distinguish them.

---

## CORS vs missing infrastructure: diagnose before patching

CORS errors and 500s that surface as network errors are the two most common masks for missing infrastructure. Never touch CORS config until you have ruled out a missing DynamoDB table or a misconfigured App Runner environment.

**Decision tree:**

1. Open the browser network tab. Note the exact HTTP status of the failing request.
2. If the request never completes (network error / blocked), check whether the response is actually a 500 — unhandled backend exceptions bypass middleware and cause the browser to report a CORS or network error instead of a status code.
3. Check App Runner logs *before forming any hypothesis*. The backend log shows the cause; the browser console shows a symptom.
4. If App Runner logs show `ResourceNotFoundException` or `ValidationException: The table does not have the specified index`, the problem is a missing DynamoDB table or GSI. Stop. Fix the infrastructure. The CORS config is correct.
5. If App Runner logs show `No Access-Control-Allow-Origin`, the CORS config is the problem. Check `CORS_ORIGINS` in `backend/apprunner.yaml` — it must be a JSON array with `https://` prefixes: `["https://app.example.com"]`.

**The canonical trap.** A new DynamoDB table declared in code but absent from CloudFormation (or declared in CloudFormation but with the stack never updated) produces a 500. The 500 triggers browser CORS handling. The engineer patches CORS. The underlying missing table never gets created. Run `aws dynamodb list-tables` and diff against all `settings.table_name(...)` calls in the service layer before touching any CORS config.

---

## TanStack Query consumer: the three-branch guard

Every TanStack Query consumer in a Next.js frontend must differentiate loading, error, and empty states. Collapsing these three into a single branch produces skeleton spinners that never resolve, silent failures, and misdirected debugging sessions.

```typescript
const { data, isLoading, isError } = useQuery({ queryKey: [...], queryFn: ... });

if (isLoading) return <Skeleton />;
if (isError) return <ErrorState onRetry={() => refetch()} />;
if (!data || data.items.length === 0) return <EmptyState />;
// data is guaranteed non-null here
```

**Rules:**

- Never render a skeleton unconditionally — gate it on `isLoading`, not on `!data`. After a successful load, `!data` may still be truthy if the API returns null; that is an empty state, not a loading state.
- Every mutation handler must surface both success and error paths to the user. Silent success and silent failure are the same from the user's perspective — both are invisible.
- After every successful mutation, call `queryClient.invalidateQueries` for all keys that could be stale. Do not rely on TTL expiry for correctness after writes.
- Pagination defaults bite detail routes: if a detail page is sourced from a paginated list query without a by-ID lookup, it silently 404s for items past page one. Add a dedicated entity-by-ID endpoint.

---

## App Runner operational debugging

App Runner's operational signals are shallow unless you know where to look.

### Health check as a diagnostic signal

Set the health check to HTTP with path `/health`, interval 10 s, timeout 5 s, unhealthy threshold 5. The TCP default will not detect an unhealthy FastAPI process.

If App Runner logs show only health-check requests with no other traffic, the frontend is not reaching the backend. Check in order:

1. `NEXT_PUBLIC_API_URL` in `frontend/.env.production` — is it the current App Runner custom domain URL?
2. Amplify deploy status — is the frontend still deploying with the old URL?
3. CORS config in `backend/apprunner.yaml` — does `CORS_ORIGINS` include the frontend origin?

### CloudWatch log recipe

Stream App Runner logs to CloudWatch automatically via the service configuration. To tail a single deployment:

```bash
aws logs tail /aws/apprunner/<service-name>/<service-id>/application \
  --since 10m --follow --region <region>
```

Filter for errors only:

```bash
aws logs filter-log-events \
  --log-group-name /aws/apprunner/<service-name>/<service-id>/application \
  --filter-pattern "ERROR" \
  --start-time $(date -d '30 minutes ago' +%s000) \
  --region <region>
```

### Python 3.11 revised-build trap

App Runner Python 3.11 uses a "revised build" that discards packages installed in the `build` phase. The symptom is `No module named uvicorn` (or any dependency) at startup — the service exits immediately, health checks fail, and App Runner marks the deployment unhealthy.

Fix: move all `pip3 install` commands from `build` to `pre-run` in `apprunner.yaml`. Use `python3 -m uvicorn` instead of `uvicorn` directly — Pip-installed scripts are not on PATH in the revised build.

### VPC Connector and the 401 cascade

Attaching a VPC Connector routes ALL App Runner outbound traffic through the VPC. Private subnets have no internet access by default. Without a NAT Gateway, App Runner cannot reach the Cognito JWKS endpoint, so every token verification fails with 401. The symptom looks like an auth bug; the root cause is a missing NAT Gateway. Verify with:

```bash
# From App Runner, attempt to reach the Cognito JWKS endpoint
# Confirm logs show connection refused or timeout, not auth failure
aws logs filter-log-events \
  --log-group-name /aws/apprunner/.../application \
  --filter-pattern "JWKS" --region <region>
```

### IAM instance role not appearing in Console

The trust policy must use `tasks.apprunner.amazonaws.com` as the principal — with the `tasks.` prefix. Without it, the role does not appear in the App Runner instance role dropdown.

---

## Half-built feature wiring audit

Shipping a feature whose write path exists but whose read, notification, or instrumentation paths do not is a recurring failure mode. Run this audit before marking any feature complete.

**Six questions to answer for every feature:**

1. **Write path exists** — does every mutation endpoint write all required fields?
2. **Read path exists** — does every consumer that needs the data have a query that fetches it?
3. **Notification path exists** — if the feature promises "user X will be notified," does the notification fire from the write path and is it tested end-to-end?
4. **Instrumentation path exists** — if the feature adds a new access path to a resource, does that path call the analytics write path? Every new access path must be instrumented in the same change.
5. **Infrastructure exists** — does every DynamoDB table, GSI, Aurora column, and IAM permission referenced by the feature exist in the live environment?
6. **Entry points verified** — run a post-deploy smoke test signed in as the most complex role combination the feature targets, click every entry point, and confirm the destination renders correctly.

**Analytics-specific wire check.** Frontend tests that assert "the tracking call fired" without asserting the on-the-wire payload are silent lies. A tracking call with an empty or malformed payload records nothing. Assert the exact payload shape in tests.

**Duration metric trap.** A polling accumulator (e.g., a 30-second interval) records zero duration for all visits shorter than the interval. Use an event-driven accumulator: record `activeStartedAt` when the user enters active state; accumulate `now - activeStartedAt` on leaving (visibility hidden, idle timeout, unmount, pagehide).

---

## Deployed-vs-local divergence

Features that pass all local tests and fail in production share a common failure mode: local mocks do not exercise infrastructure existence.

**Checklist:**

| Check | Local behavior | Production behavior |
|-------|---------------|---------------------|
| DynamoDB table exists | moto mock accepts any table name | missing table throws `ResourceNotFoundException` |
| GSI exists | moto mock accepts any index name | missing GSI throws `ValidationException` |
| Cognito JWT verification | `COGNITO_VERIFY_TOKENS=false` bypasses JWKS | live tokens require reachable JWKS endpoint |
| Env vars present | `.env.local` or test defaults | `apprunner.yaml` `run.env` only; Console vars ignored in REPOSITORY mode |
| Aurora enabled | `AURORA_ENABLED=false` — Aurora client never initialized | `AURORA_ENABLED=true` — connection failure at startup crashes the service |

**The debugging move for any "works locally, fails in prod" bug:**

1. Check App Runner environment variables against `backend/apprunner.yaml` — the live env is exactly what is in that file, nothing more.
2. Run `aws dynamodb list-tables` and diff against all table references in code.
3. Check whether any GSI added to the CloudFormation template was applied via `aws cloudformation update-stack`. Template changes do not self-apply.
4. Confirm `CORS_ORIGINS` includes the exact frontend origin with `https://` — not a wildcard, not missing the scheme.
5. If Auth fails in prod but not locally, confirm the App Runner VPC Connector has a NAT Gateway route to the public internet.

---

## Cross-references

- `core/practices/debugging.md` — the five-phase Observe → Hypothesize → Verify → Plan → Implement protocol. Read before any debugging session.
- `core/practices/anti-patterns.md` — minimal blast radius, trial-and-error prohibition, single authoritative schema.
- `practices/deployment-aws.md` — App Runner setup, Amplify SSR detection, env var placement, post-deploy smoke test.
- `practices/infra-aws.md` — DynamoDB multi-table design, GSI deployment trap, analytics aggregate item pattern.
- `practices/security-aws.md` — Cognito JWT `Depends()` patterns, role-based access guards.
- `practices/testing-fullstack.md` — `COGNITO_VERIFY_TOKENS=false`, moto DynamoDB mocks, Playwright E2E setup.

---

<!-- Assembled from: core/practices/debugging.md §Phase 1 Observe, §Deployment and Configuration Failures, §Analytics and Instrumentation Bugs, §Feature Completeness and Verification; core/practices/anti-patterns.md §API and data layer design; core/stack-modules/full-stack-web-aws/v0.1.0/practices/deployment-aws.md §App Runner setup, §Troubleshooting table, §Environment variables, §Post-deploy smoke test; core/stack-modules/full-stack-web-aws/v0.1.0/practices/infra-aws.md §DynamoDB GSI deployment trap, §Analytics Aggregate Item. Stack module: full-stack-web-aws v0.1.0. -->
