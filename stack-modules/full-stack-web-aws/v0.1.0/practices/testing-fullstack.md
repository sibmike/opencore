# Testing: Full-Stack Web on AWS

Stack-specific testing patterns for a Python/FastAPI backend + Next.js frontend deployed on AWS.
Coverage minimums and critical-path policy live in [core/practices/testing-principles.md](../../practices/testing-principles.md).

---

## Backend test stack: pytest + pytest-asyncio + moto

All backend tests are async. Use `pytest-asyncio` in auto mode so every `async def test_*` function
runs without an explicit decorator.

`moto` mocks DynamoDB (and other AWS services) for `aioboto3` calls — no real AWS infrastructure
required in CI or local development. The mock intercepts at the `botocore` transport layer, so your
service code runs unchanged.

Three mandatory lines in `conftest.py`:

```python
import os
os.environ["COGNITO_VERIFY_TOKENS"] = "false"   # skip JWKS fetch during tests
os.environ["STEP_FUNCTION_ARN"] = ""            # empty ARN = no-op guard for async triggers
```

Set `COGNITO_VERIFY_TOKENS=false` before the app module is imported; setting it after is a no-op
because the dependency factory reads it at import time.

The `STEP_FUNCTION_ARN=""` guard is the standard local-dev pattern for any AWS async trigger
(Step Functions, SQS, SNS, EventBridge): an empty config value causes the caller to short-circuit
rather than fail with a missing-resource error.

---

## Async fixture pattern: moto + aioboto3

Wrap every DynamoDB fixture in a `mock_dynamodb()` context. Create all tables — including GSIs —
inside the context before yielding.

```python
# conftest.py
import pytest
import aioboto3
from moto import mock_dynamodb

@pytest.fixture
async def dynamo_tables():
    with mock_dynamodb():
        session = aioboto3.Session()
        async with session.resource("dynamodb", region_name="us-east-1") as dynamodb:
            await dynamodb.create_table(
                TableName="my_table",
                KeySchema=[{"AttributeName": "PK", "KeyType": "HASH"},
                           {"AttributeName": "SK", "KeyType": "RANGE"}],
                AttributeDefinitions=[{"AttributeName": "PK", "AttributeType": "S"},
                                      {"AttributeName": "SK", "AttributeType": "S"}],
                BillingMode="PAY_PER_REQUEST",
            )
            yield dynamodb
```

Every test that exercises a DynamoDB-backed route must list all tables used by that route in the
fixture. A missing table causes the test to fail for the wrong reason (table not found instead of
the logic you intended to test) — or worse, silently pass if the service catches the exception.
Audit `conftest.py` table list against all tables referenced in each service under test before
writing integration tests.

---

## boto3 response-key casing

AWS SDK response keys are **not uniformly cased** across services. Two common errors:

| Service call | Correct key | Wrong key (fails with `KeyError`) |
|---|---|---|
| `bedrock-runtime.invoke_model` | `"body"` (lowercase) | `"Body"` |
| `s3.get_object` | `"Body"` (uppercase) | `"body"` |

A test that mocks the wrong key shape will pass locally while the production call fails. To find
the authoritative key name for any service, inspect:
`botocore/data/<service>/<version>/service-2.json` → `output.members`.

When multiple AWS services run in the same pipeline, verify each service's response shape
independently before writing tests.

---

## Cognito token verification in tests

`COGNITO_VERIFY_TOKENS=false` disables the JWKS fetch and signature check inside the FastAPI
`get_current_user()` dependency. In test scope, the dependency reads the `Authorization: Bearer`
header and constructs a minimal user object from the raw (unverified) token payload.

Provide a fake Cognito sub and email in the token fixture:

```python
import jwt   # PyJWT, not python-jose

def fake_token(sub: str = "test-user-id", email: str = "test@example.com") -> str:
    return jwt.encode(
        {"sub": sub, "email": email, "custom:role": "founder"},
        key="test-secret",
        algorithm="HS256",
    )
```

Never set `COGNITO_VERIFY_TOKENS=true` in a test that does not have network access to the Cognito
JWKS endpoint — it will produce flaky DNS failures, not meaningful test failures.

---

## Backend fixture layout

```
backend/tests/
  conftest.py                        # aioboto3 + moto wiring; COGNITO_VERIFY_TOKENS=false
  fixtures/
    bedrock_responses/               # JSON stubs for Bedrock InvokeModel responses
      claude_memo_response.json
      claude_suggestion_response.json
    dynamo_seed.py                   # helpers that create typed items in moto-mocked tables
```

Keep Bedrock response stubs as static JSON. When the response schema changes, update the stub
first, then fix the affected service — this preserves the "stub is ground truth" invariant.

---

## Frontend test stack: Vitest + React Testing Library + MSW v2

Three tools, each with a distinct responsibility:

- **Vitest** — Vite-native TypeScript runner. Shares the same module resolution as the app;
  no separate Babel config required.
- **React Testing Library** — assert on what the user sees and interacts with, not on component
  internals or implementation details.
- **MSW v2** — intercepts `fetch` at the network level inside the test process (no proxy). MSW
  errors on unhandled requests by default: a test that forgets a handler gets an explicit failure,
  not a silent empty response.

Run tests with:
```bash
npm test                  # watch mode
npm run test:coverage     # coverage report
```

---

## Frontend test infrastructure

```
src/test/
  setup.ts                           # imported by vitest.config.ts; starts MSW server
  mocks/
    handlers/
      <domain>.ts                    # one handler file per API domain
    factories.ts                     # typed factory functions for domain objects
  helpers/
    error-responses.ts               # withErrorResponse() for per-test error overrides
    render-with-providers.tsx        # renderWithProviders() utility
```

**`renderWithProviders()`** wraps the component under test in a `QueryClientProvider` configured
with `retry: false`. Without `retry: false`, TanStack Query will retry failed requests three times
before the test resolves, making error-path tests slow and non-deterministic.

**`withErrorResponse()`** overrides a specific handler for a single test:

```typescript
it('shows error state on 500', async () => {
  withErrorResponse(server, 'GET', '*/api/v1/resource/:id', 500);
  render(<MyComponent />);
  expect(await screen.findByText('Something went wrong')).toBeInTheDocument();
});
```

**Factory functions** in `factories.ts` produce typed objects with sensible defaults. Accept a
partial override argument so each test can vary only the fields it cares about:

```typescript
export function makeResourceDetail(overrides: Partial<ResourceDetail> = {}): ResourceDetail {
  return { id: 'r-001', name: 'Test Resource', status: 'active', ...overrides };
}
```

---

## MSW v2 handler pattern

One handler file per API domain. Export a named array; the central `setup.ts` spreads all arrays
into the MSW server:

```typescript
// test/mocks/handlers/resource.ts
import { http, HttpResponse } from 'msw';
import { makeResourceDetail } from '../factories';

export const resourceHandlers = [
  http.get('*/api/v1/resource/:id', ({ params }) => {
    return HttpResponse.json(makeResourceDetail({ id: params.id as string }));
  }),

  http.post('*/api/v1/resource', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json(makeResourceDetail(body as Partial<ResourceDetail>), { status: 201 });
  }),
];
```

Keep handler paths as glob patterns (`*/api/v1/...`) so they match regardless of the base URL used
in the test environment.

---

## TanStack Query: mandatory three-branch guard

Every component that consumes a TanStack Query hook must handle three states explicitly:

```tsx
const { data, isLoading, isError } = useResource(id);

if (isLoading) return <LoadingSkeleton />;
if (isError)   return <ErrorMessage error={error} />;
if (!data)     return <EmptyState />;  // or redirect

return <ResourceView data={data} />;
```

Collapsing `isError` into `isLoading || !data` produces a perpetual skeleton when the request
returns a 404 or network error — the skeleton never resolves because neither condition clears.
Treat error and empty as distinct states.

---

## E2E testing with Playwright

E2E tests require both the Next.js frontend and FastAPI backend running. Config lives at
`frontend/e2e/playwright.config.ts`; set `PLAYWRIGHT_BASE_URL` for CI or remote environments.

Define three critical user-journey tests at minimum:

1. **Onboarding**: sign up → complete required setup step → land on authenticated home page.
2. **Upload + poll**: trigger an upload → wait for async processing to complete → assert the
   result is visible (exercises the exponential-backoff polling path).
3. **Shared view**: visit a share link as an unauthenticated user → verify content renders →
   verify gated actions are blocked.

Run the smoke suite post-deploy via `PLAYWRIGHT_BASE_URL=https://your-domain npx playwright test`
and store JUnit reports to S3 with a lifecycle policy for auditability.

---

## Playwright screenshot pipeline for landing pages

Use Playwright to capture marketing screenshots from mock Next.js pages rather than production
pages. Mock pages live at `frontend/src/app/screenshots/<name>/page.tsx` and render with hardcoded
fixture data so screenshots are deterministic.

**Adding a mock page:**

1. Create `frontend/src/app/screenshots/<name>/page.tsx`.
2. Export a default component using hardcoded mock data — no API calls.
3. Use a `MockTopBar` pattern (copy from an existing screenshot page and adjust nav items).
4. Apply responsive classes (`hidden md:flex`, `grid-cols-1 sm:grid-cols-3`) so the page looks
   correct at both desktop and mobile capture sizes.
5. Add the page entry to the capture-script pages array with an initial `clip.height`.

**Capture parameters:**

| Parameter | Desktop | Mobile |
|---|---|---|
| `viewport.width` | 1280 | 375 |
| `viewport.height` | 900 | 812 |
| `clip.height` | 780–820 (tune per page) | 812 |
| `colorScheme` | `'dark'` | `'dark'` |
| `waitUntil` | `'networkidle'` | `'networkidle'` |

Remove the Next.js dev overlay before capture:

```typescript
for (const sel of ['nextjs-portal', '[data-nextjs-dialog-overlay]', '[data-nextjs-toast]']) {
  await page.evaluate((s) => document.querySelectorAll(s).forEach(el => el.remove()), sel);
}
```

Tune `clip.height` per page: increase to capture more content, decrease to crop tighter.
The dev overlay removal handles the top-of-page remnant; the clip handles the bottom.

---

## ScreenshotWithPhone and PhoneMockup components

For landing-page sections that need a simultaneous desktop + mobile illustration, compose
`ScreenshotWithPhone`:

```tsx
function ScreenshotWithPhone({ src, alt, mobileSrc, mobileAlt }) {
  return (
    <div className="relative mb-16">
      <ScreenshotFrame src={src} alt={alt} />
      <div className="hidden md:block absolute -bottom-12 right-6 w-[140px] lg:w-[160px] rotate-1 z-10">
        <PhoneMockup src={mobileSrc} alt={mobileAlt} />
      </div>
    </div>
  );
}
```

The phone is hidden on mobile viewports (`hidden md:block`); `mb-16` on the parent accommodates
the overflow. The `rotate-1` tilt is purely cosmetic.

`PhoneMockup` (`frontend/src/components/PhoneMockup.tsx`) is a pure-CSS iPhone-style device frame:
- Outer bezel: `rounded-[2.5rem] border-[6px] border-zinc-800`.
- Dynamic Island: centered pill at top.
- Screen area: `rounded-[2rem] overflow-hidden` with a Next.js `Image`.
- Props: `{ src: string; alt: string; className?: string }`.

---

## Git worktrees: frontend vs backend constraints

In a Next.js + Python monorepo, worktrees behave differently per layer.

**Frontend — do not run tests from a worktree.**
`node_modules/` is absent in a worktree's `frontend/` directory. Symlinking or junctioning it
does not work reliably on Windows. Running `npm test` or `npx vitest` from the worktree produces
`MODULE_NOT_FOUND`. Always run frontend tests from the main repo's `frontend/` directory. New
frontend test files written in a worktree are testable only after merging to the integration branch.

**Backend — tests can run from a worktree.**
Python resolves the interpreter via the venv path, not a `node_modules`-style relative directory.
Invoke pytest using the absolute venv path:

```bash
backend/.venv/Scripts/python.exe -m pytest tests/ -v
```

The worktree carries its own `backend/` directory with new test files, so they are immediately
testable without merging.

---

## Test mocking discipline

Never make real service calls in unit or component tests. The contract is:

| Layer | What to mock | How |
|---|---|---|
| FastAPI (DynamoDB) | All table access | `moto` + `mock_dynamodb()` fixture |
| FastAPI (Bedrock, S3) | All service calls | Stub responses in `fixtures/bedrock_responses/` |
| FastAPI (Cognito) | JWT verification | `COGNITO_VERIFY_TOKENS=false` in `conftest.py` |
| Next.js (API) | All `fetch` calls | MSW v2 handlers |
| Next.js (Auth) | Token fetch | Vitest mock for `Amplify.Auth.fetchAuthSession` |

A test that reaches a real AWS endpoint in CI is not a unit test — it is an unintentional
integration test with undeclared network dependencies and non-deterministic failure modes.

Assertion discipline: when a test verifies that a write happened, assert `> 0` or `== expected`,
never `>= 0`. An assertion of `>= 0` is always true even when the write silently targeted the
wrong table or key.

---

## Cross-references

- [core/practices/testing-principles.md](../../practices/testing-principles.md) — coverage minimums, critical-path 100% rule
- [core/practices/anti-patterns.md](../../practices/anti-patterns.md) — generic test anti-patterns
- [practices/conventions-fullstack.md](./conventions-fullstack.md) — TanStack Query patterns (polling, cache invalidation, key factories)
- [practices/debugging-fullstack.md](./debugging-fullstack.md) — wiring failures caught at test time
- [practices/deployment-aws.md](./deployment-aws.md) — `PLAYWRIGHT_BASE_URL` + smoke test post-deploy
- [practices/infra-aws.md](./infra-aws.md) — Bedrock response shapes, DynamoDB table patterns

<!--
Assembled from:
  F-040 (dev-environment.md / Known Pitfalls — COGNITO_VERIFY_TOKENS, Step Functions no-op guard)
  F-045 (ERRATA.md / ERR-004 — boto3 response-key casing)
  F-108 (post-deploy-checklist.md / Debugging-Specific Addendum — test mocking contract)
  F-118 (README.md / ## Testing — pytest + vitest commands)
  F-120 (screenshots.md / Adding a new mock page)
  F-121 (screenshots.md / Key parameters — viewport, colorScheme, waitUntil)
  F-122 (screenshots.md / Adjusting crop height)
  F-123 (screenshots.md / Desktop + Mobile: ScreenshotWithPhone)
  F-124 (screenshots.md / PhoneMockup component)
  F-130 (testing.md / Stack: pytest + pytest-asyncio + moto)
  F-131 (testing.md / Test Fixtures — fixture layout)
  F-132 (testing.md / Key Pattern: Async Fixtures with Moto)
  F-133 (testing.md / Stack: Vitest + React Testing Library + MSW v2)
  F-134 (testing.md / Test Infrastructure — MSW handler layout, renderWithProviders)
  F-135 (testing.md / Key Pattern: MSW Handler)
  F-136 (testing.md / E2E Testing — Playwright stubs, config path)

Stack doc: full-stack-web-aws v0.1.0
-->
