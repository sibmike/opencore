# Code Conventions: Full-Stack Web (AWS)

Code conventions for the Python/FastAPI + TypeScript/Next.js + AWS archetype. Apply uniformly across all layers; deviation requires an explicit rationale in a design doc.

---

## Backend conventions (Python / FastAPI)

**Validation:** use Pydantic models at every request and response boundary — no raw `dict` passing across layers. Validate on input, serialize on output.

**Async discipline:** use `aioboto3` for all AWS SDK calls inside async handlers. Never import `boto3` (sync) in an async context — it blocks the event loop and degrades throughput under load.

**Database access:** no ORM. Drive DynamoDB directly with `put_item` / `get_item` / `query` / `scan` and serialize results via Pydantic. Keep access patterns close to the table definition; anything that requires more than one round-trip belongs in a service function, not a router.

**URL shape:** prefix every route `/api/v1/`. Return all JSON fields in `snake_case`. The frontend `apiClient` is responsible for camelCase conversion at the boundary — keep the wire format stable.

**Error propagation:** raise domain exceptions (`NotFoundError`, `ForbiddenError`, `ConflictError`, etc.) from services. A single global exception handler in `app/main.py` maps them to HTTP status codes. Services that raise `HTTPException` directly break this contract and make the domain logic untestable in isolation.

**Ownership checks:** implement a single reusable dependency (`get_owned_<resource>`) that combines JWT verification with a DynamoDB ownership query. Repeat the same verification inline in multiple routes and they will drift.

**Layering rule:** services import from no other service. Routers import services. No service imports a router. Pydantic validation sits at the router boundary, not inside service functions.

**Static analysis:** run `ruff format` and `ruff lint` in CI. Enable Python type hints on every function signature. Failing the linter blocks merge.

---

## Frontend conventions (TypeScript / Next.js)

**Strict mode:** set `"strict": true` in `tsconfig.json`. Never disable strict checks per-file — every suppression is a future bug waiting to surface.

**Component library:** use `shadcn/ui` for all UI primitives. Add new components via the shadcn CLI rather than hand-rolling. `Lucide React` for all icons — no mixed icon libraries.

**Styles:** Tailwind CSS v4 with design tokens declared as `@theme inline` in `globals.css`. Avoid one-off inline styles or arbitrary values where a token applies.

**API state:** all server data lives in TanStack Query. Never reach for `useState` to cache a fetched value, and never store server responses in React context or Zustand. See [TanStack Query patterns](#tanstack-query-patterns) for structural rules.

**Local UI state:** `useState` for transient UI state (open/closed, hover, selected tab). `useContext` for shared UI config (theme, locale). Neither one is appropriate for server data.

**API boundary:** pass all requests through a single `apiClient` helper that calls `Amplify.Auth.fetchAuthSession()`, injects the `Bearer` token, and auto-converts `snake_case` responses to `camelCase`. Manual field mapping in individual hooks is a layering violation.

**TypeScript:** every API response has a TypeScript interface. Every hook returns typed data. `any` is forbidden; use `unknown` with a type guard when the shape is genuinely variable.

**Server components:** default to React Server Components in the Next.js App Router. Move to a Client Component only when the component needs browser APIs, event handlers, or TanStack Query hooks.

**Static analysis:** run `eslint` and `npx tsc --noEmit` in CI. Both must pass before merge.

---

## TanStack Query patterns

Four sub-patterns compose the full data-fetching contract. Use all four consistently; partial adoption causes cache incoherence.

### Query key factories

Each hook file exports a typed `keys` object. Derive all `queryKey` values from it — never inline string literals.

```typescript
export const resourceKeys = {
  all: ['resources'] as const,
  detail: (id: string) => ['resources', id] as const,
};
```

Centralising keys in one place means `invalidateQueries` and `prefetchQuery` callsites are impossible to mismatch. A key change propagates automatically to every consumer.

### Tiered staleTime

Match `staleTime` to data volatility:

| Volatility | Example | staleTime |
|---|---|---|
| High (real-time) | processing status, notifications | `0` |
| Medium (session) | resource lists, user profile | `60_000` (1 min) |
| Low (static) | config, pricing tiers | `300_000` (5 min) |

Set `staleTime` at the `useQuery` call, not globally. Global defaults hide per-query intent.

### Cache invalidation

Call `queryClient.invalidateQueries({ queryKey: resourceKeys.all })` inside the mutation's `onSuccess` callback. This pattern ensures the UI reflects server state immediately after a write without an explicit refetch in the component.

```typescript
useMutation({
  mutationFn: (payload) => apiClient.post('/api/v1/resources', payload),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: resourceKeys.all });
  },
});
```

Also wire `onError` to a visible toast. A mutation that fails silently is a UX bug.

### Exponential-backoff polling

Use for long-running async jobs (AI processing, file pipelines, etc.). Start at 3 s, double each interval, cap at 30 s, apply a 5-minute absolute timeout.

```typescript
const [pollInterval, setPollInterval] = useState(3000);
const startedAt = useRef(Date.now());

useQuery({
  queryKey: resourceKeys.detail(jobId),
  refetchInterval: (data) => {
    if (data?.status === 'complete' || data?.status === 'error') return false;
    if (Date.now() - startedAt.current > 5 * 60 * 1000) return false;
    const next = Math.min(pollInterval * 2, 30_000);
    setPollInterval(next);
    return pollInterval;
  },
});
```

The FastAPI endpoint returns `Retry-After` (header) and `estimated_completion_at` (body field) whenever status is still in-progress. Surface `estimated_completion_at` in the UI loading state.

**Stop conditions:** stop polling on `status=complete`, `status=error`, component unmount, or the 5-minute timeout. Every polling query must have an explicit terminal condition — an open-ended `refetchInterval: 3000` is always a bug.

**Three-branch guard:** every TanStack Query consumer must handle three branches explicitly:

```typescript
if (isLoading) return <Skeleton />;
if (isError) return <ErrorState error={error} />;
if (!data) return null;
// safe to render data here
```

Collapsing `isError` into the loading branch causes a perpetual skeleton on network failure. The collapsed pattern has shipped silent production bugs repeatedly; treat it as a linting violation.

---

## Naming conventions

Consistency across layers eliminates cognitive overhead when navigating between backend, frontend, and infrastructure code.

| Layer | Convention | Example |
|---|---|---|
| Backend Python files | `snake_case` | `resource_service.py` |
| Backend service class | `{Entity}Service` | `ResourceService` |
| FastAPI router files | `snake_case` | `resources.py` |
| API endpoint paths | `snake_case` segments | `/api/v1/resources/{id}/publish` |
| API JSON fields | `snake_case` | `resource_id`, `ai_score` |
| Frontend files | `kebab-case` | `use-resource.ts`, `resource-card.tsx` |
| React components | `PascalCase` | `ResourceCard`, `UploadDialog` |
| TypeScript fields | `camelCase` | `resourceId`, `aiScore` |
| TanStack Query hook files | `use-{entity}.ts` | `use-resources.ts` |
| TanStack Query key exports | `{entity}Keys` | `resourceKeys` |
| CSS variables | `kebab-case` | `--muted-foreground` |
| DynamoDB PK/SK values | `UPPER#value` | `USER#abc`, `RESOURCE#xyz` |
| Test files | source name + `.test` | `use-resource.test.tsx` |

The `apiClient` converts `snake_case` responses to `camelCase` automatically. Never manually rename fields at the component level — it creates two sources of truth.

---

## Anti-patterns to avoid

These patterns have caused production bugs in this archetype. Treat each as a hard rule, not a suggestion.

**Sync `boto3` in async handlers.** Blocks the event loop. Every database call inside a FastAPI async route must use `aioboto3`.

**`HTTPException` raised from service functions.** Services are HTTP-agnostic. Raise `NotFoundError`, `ForbiddenError`, etc. from services; let the global handler convert them. A service that raises `HTTPException` cannot be tested without spinning up an ASGI app.

**Data fetching outside TanStack Query hooks.** Calling `fetch()` or `apiClient` inside a `useEffect` bypasses cache management, deduplication, and error handling. All API calls live in `useQuery` or `useMutation`.

**Global state for server data.** React context and Zustand are for shared UI config, not for API responses. Server data in global state goes stale silently and breaks back-navigation.

**Duplicate validation between TypeScript and Pydantic.** Define the contract once in Pydantic on the backend. The TypeScript interface at the boundary mirrors it but does not re-validate business rules. Keeping business logic in both layers guarantees drift.

**Optional-chained props threaded from page root without explicit verification.** A prop passed as `resource?.field` through three layers of components becomes a silent no-op if the chain is never populated. Thread props explicitly or use a context; audit the full prop path before shipping a feature.

**Paginated list lookups for single-item action URLs.** If an action requires a specific item (e.g., a share URL), fetch it by primary key with a direct `GetItem` — not by scanning page 1 of a list endpoint. Items past the first page are silently missing.

---

## Cross-references

- `practices/testing-fullstack.md` — `COGNITO_VERIFY_TOKENS=false`, moto async fixture, MSW v2 handler pattern.
- `practices/debugging-fullstack.md` — TanStack Query three-branch guard failure modes, CORS vs missing DynamoDB tables.
- `practices/infra-aws.md` — DynamoDB key design, versioned config pattern, analytics event instrumentation.
- `practices/security-aws.md` — `get_current_user()` Depends(), `require_any_role()` factory, JWT discipline.
- `core/practices/anti-patterns.md` — generic anti-patterns not tied to this stack.
- `core/practices/architecture-principles.md` — generic layering and separation-of-concerns principles.

---

<!-- PROVENANCE
Stack module: full-stack-web-aws v0.1.0
Generated from fragments: F-008, F-009, F-014, F-016, F-017, F-018, F-019, F-020, F-021, F-022, F-023
Source documents: architecture.md (F-008, F-009), CLAUDE.md (F-014), conventions.md (F-016 – F-023)
YAML depth reads: split-reports/stack-extraction/manifest/per-doc/conventions.yaml
Synthesis date: 2026-05-10
-->
