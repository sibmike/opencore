# Security: AWS Full-Stack Web App

> Cognito JWT auth, role-based FastAPI dependencies, Google SSO, pre-signed URL discipline, secrets management, and known gaps. Companion to `core/practices/security-principles.md`.

---

## Cognito Auth Flow

Cognito issues JWTs. Every protected FastAPI route validates the token via a `Depends()` chain.

**Backend dependency chain in `dependencies.py`:**

1. `get_current_user()` — fetches the Cognito JWKS endpoint, verifies the token signature and expiry, and extracts `user_id` from claims. Raise 401 on any failure; never fall through to unauthenticated access.
2. `get_owned_resource()` — performs a DynamoDB GSI lookup to confirm the authenticated user owns the requested item. Raise 403 if ownership check fails.
3. `get_db()` — yields an aioboto3 DynamoDB resource via `Depends()` for testability.

**Frontend token injection:**

Call `Amplify.Auth.fetchAuthSession()` before every API request. The `apiClient()` wrapper attaches the resulting Bearer token to every outbound `Authorization` header. No manual token storage in `localStorage` — Amplify v6 handles refresh automatically.

**Test flag:** Set `COGNITO_VERIFY_TOKENS=false` in `conftest.py` to bypass JWKS during unit and component tests. Never set this flag in production or staging.

**Cognito user pool settings to configure explicitly:**

| Setting | Value |
|---|---|
| Sign-in attribute | Email only |
| Password policy | 8+ characters, mixed case, number |
| Custom attribute | `custom:role` — `"founder"` or `"investor"` |
| Access token expiry | 1 hour |
| Refresh token expiry | 30 days |
| MFA | Off by default — plan an opt-in upgrade path |

Two distinct sign-up flows share one pool. The `custom:role` attribute carries the user's primary role; multi-role users need `require_any_role()` (see below).

**User pool ID format:** `<region>_<poolId>` — NOT an ARN. The Client ID is a separate alphanumeric string. Both are public values safe to embed in `NEXT_PUBLIC_*` env vars. Passing `undefined` for either causes `Failed to construct 'URL': Invalid URL` at runtime — verify both are set before any auth call.

---

## Role-Based FastAPI Dependencies

Use a dependency factory, not ad-hoc role checks scattered across routes.

```python
# dependencies.py
def require_any_role(*roles: str):
    async def checker(user: User = Depends(get_current_user)):
        if user.role not in roles:
            raise HTTPException(status_code=403)
    return Depends(checker)

# Usage
@router.post("/analysis")
async def trigger_analysis(
    _: None = require_any_role("founder", "admin"),
    db: DB = Depends(get_db),
):
    ...
```

Typed named dependencies (`FounderUser`, `InvestorUser`, `AdminUser`) make route signatures self-documenting. Dual-role endpoints pass comma-separated roles to the factory.

**Read-only viewer pattern:** Add a `viewer_role` field (`"owner"` / `"member"` / `"investor"`) to every resource detail response. A `WritableResource` dependency wraps `OwnedResource` and raises 403 when `viewer_role == "investor"`. Apply `WritableResource` to all write routes — upload, update, delete, settings, share link, members. The frontend reads `viewer_role` via a context flag (`isReadOnlyView`) and hides write controls accordingly.

**Auth gate for deep links:** When `useResource()` returns a 401, render an auth-gate component with a `?redirect=<current-path>` parameter. On 403, render a dedicated access-denied page. Never silently no-op on auth failures.

---

## Permission Service (Aurora-Backed)

Relational permission resolution lives in a single `permission_service.py`. No other module imports `asyncpg`.

Resolution priority order for resource access:

1. Team membership (members table joined on resource ID — owner/member role) → full access
2. Resource visibility tier (public/unlisted/private × open/subscription/invite-only)
3. Viewer qualification (explicit grant, unlock, reviewer assignment, subscription, free tier)

Document-level visibility uses per-viewer overrides in a sparse Aurora table, capped by tier.

Guard `AURORA_ENABLED=false` (default `false`) so the app starts without Aurora in local dev and during initial deploy. All permission functions return graceful fallbacks when the flag is off.

---

## Google SSO via Cognito Federated IdP

This is a one-time console wiring — no Lambda, no CloudFormation, no CI/CD.

**Step 1 — Google Cloud Console.**
Create an OAuth client ID (Web Application type). Set the authorized redirect URI to exactly `https://<COGNITO_DOMAIN>/oauth2/idpresponse`. Leave authorized JavaScript origins empty (Cognito handles the OAuth redirect, not the browser).

For internal-only SSO (company domain only), set OAuth consent screen User Type to **Internal** — Google rejects non-domain users at its edge before your app sees anything. No Pre-Sign-up Lambda required.

**Step 2 — Cognito hosted-UI domain.**
Set a domain prefix under App integration → Domain. The resulting URL (`<prefix>.auth.<region>.amazoncognito.com`) becomes `NEXT_PUBLIC_COGNITO_DOMAIN`.

**Step 3 — Add Google as an IdP.**
Sign-in experience → Federated identity provider sign-in → Add Google. Paste the Client ID and Client secret. Scopes: `profile email openid`. Attribute map:

- `email` → `email`
- `email_verified` → `email_verified`
- `username` → `sub`

If `email_verified` is not mapped, backend auth checks always see `False` — this is a silent breakage that survives smoke tests unless you explicitly verify the claim value.

**Step 4 — Update the app client.**
App integration → App clients → Hosted UI → Edit. Add callback and sign-out URLs. Enable both `Cognito user pool` and `Google` identity providers. Set grant type to `Authorization code grant`.

**Step 5 — Amplify env vars.**
Set three `NEXT_PUBLIC_*` vars in Amplify Console (not in `.env.production` — these rarely change and are not secrets):

| Variable | Example value |
|---|---|
| `NEXT_PUBLIC_COGNITO_DOMAIN` | `myapp-prod.auth.us-east-1.amazoncognito.com` |
| `NEXT_PUBLIC_OAUTH_REDIRECT_SIGNIN` | `https://myapp.com/login` |
| `NEXT_PUBLIC_OAUTH_REDIRECT_SIGNOUT` | `https://myapp.com/` |

Trigger a redeploy after setting. These are public values — they do not need secrets rotation.

**Step 6 — Smoke test.**
Open an incognito window. Navigate to an auth-gated page. Click the SSO button. Verify the `?code=...` callback completes. Sign out and confirm redirect. A `redirect_mismatch` error means the callback URL in Step 4 and `NEXT_PUBLIC_OAUTH_REDIRECT_SIGNIN` differ by even one character — trailing slash, `http` vs `https`, or extra path segment.

**What Google SSO does not handle:**
- Account linking: a user who registered with email/password and later signs in via Google gets two separate Cognito sub IDs. Plan for this if your app must deduplicate accounts.
- Automated off-boarding: federated users accumulate in Cognito. Add a periodic cleanup job if you need to remove users who never accessed the app.
- Role enforcement: Google SSO grants Cognito access, not application roles. Backend `require_admin` (or equivalent) must independently check `email_verified` and the email domain — Google login alone is not sufficient authorization.

---

## Pre-Signed URL Discipline (S3)

Pre-signed URLs delegate upload and download authority to the client without exposing AWS credentials. Three disciplines apply.

**Upload flow:**
1. Backend creates a DynamoDB record (`status=uploading`) and issues a pre-signed `PUT` URL with a `content-length-range` condition.
2. Client uploads directly to S3 via `PUT`.
3. Client calls a confirm endpoint; backend updates status and starts the async pipeline.

Never skip the confirm step. Without it, the DynamoDB record stays in `uploading` state and the pipeline never runs.

**Three-layer size enforcement:** Apply file size limits at three independent layers — frontend client-side rejection before the presign request, FastAPI/Pydantic validation on the presign endpoint, and the S3 `content-length-range` condition on the presigned URL. If all three layers do not agree on the same limit, users get inconsistent error messages and one layer will silently accept what another rejected.

**Presigned URL expiry:** Default to 1 hour for document downloads. Use shorter expiry (5–15 minutes) for sensitive materials. S3 bucket must have public access blocked and a TLS-only bucket policy — never issue direct public URLs. Audit expiry values when adding new resource types (S5 gap — currently hardcoded).

**Document viewing via presigned URLs:** Resolve fresh presigned URLs at view time from a DynamoDB document record, not from a cached field. Pre-signed URLs expire; cached values silently 403 after expiry.

---

## Secrets Management

Separate secrets from public config. The boundary: if the value enables access to AWS resources or databases, it is a secret.

**Required secrets** (never in frontend code, never in version control):

| Variable | Purpose |
|---|---|
| `AWS_ACCESS_KEY_ID` | AWS API access (backend + CI/CD) |
| `AWS_SECRET_ACCESS_KEY` | AWS API access (backend + CI/CD) |
| `COGNITO_USER_POOL_ID` | JWT verification (backend) |
| `COGNITO_CLIENT_ID` | Auth client config (frontend + backend) |
| `S3_BUCKET_NAME` | File upload/download (backend) |
| `AURORA_HOST` | Aurora PostgreSQL endpoint (backend) |
| `AURORA_PASSWORD` | Aurora DB password — store in Secrets Manager, inject at runtime |
| `AURORA_ENABLED` | Feature flag for permission service |

**Public config** (safe in `NEXT_PUBLIC_*` env vars):
- `NEXT_PUBLIC_COGNITO_USER_POOL_ID`
- `NEXT_PUBLIC_COGNITO_CLIENT_ID`
- `NEXT_PUBLIC_COGNITO_IDENTITY_POOL_ID`
- `NEXT_PUBLIC_API_URL`
- `NEXT_PUBLIC_COGNITO_DOMAIN`
- OAuth redirect URLs

**Account deletion tokens:** Store 6-digit verification codes and request tokens as transient fields on the user DynamoDB item (`deletion_code`, `deletion_request_token`, `deletion_code_expires_at`). Clear them before the cascade deletion to prevent replay. Overwrite on each new request rather than appending.

---

## Sharing & Passcode Security

bcrypt passcodes protect share links at the backend layer.

- Hash with `bcrypt.hashpw(passcode.encode(), bcrypt.gensalt(rounds=12))`. Never store or log the plain-text passcode.
- Validate via `X-Passcode` header on the share endpoint. Return 401 on mismatch with a generic message — do not confirm whether the code exists.
- Frontend generates the 6-character code (no ambiguous characters) and passes it once to `PATCH /share/link`. The backend stores only the hash. The frontend never stores the plain-text code after display.
- Persist the entered passcode in `sessionStorage` to avoid re-prompting on page reload within the same session. Clear on sign-out.
- Use a `ForbiddenError` with an optional machine-readable `code` parameter for access-control responses that the frontend must branch on programmatically.

---

## Account Deletion Cascade

Role-based cascades prevent orphaned data.

```
DELETE /account/confirm
  → validate code + token + expiry
  → _delete_{role}() based on user.role
    founder: query all owned resources → delete each → delete user profile LAST
    investor: delete profile, invitations, shares, Aurora records → delete user profile LAST
  → profile deletion is the cascade anchor — delete it last
```

Cognito user stays intact after application-layer deletion. `init_user()` upsert on next sign-in creates a fresh profile — this enables re-registration without a Cognito admin action.

---

## Known Security Gaps

Address these before public launch:

| # | Gap | Severity | Remediation |
|---|---|---|---|
| 1 | No rate limiting on any endpoint | **High** | Add `slowapi` middleware; configure per-route limits (auth 10/min/IP, upload 20/hr/user, AI 50/day/resource, email 50/day/resource) |
| 2 | No password reset UI | **Medium** | Cognito account recovery is configured; add a forgot-password page calling Cognito's `forgotPassword` + `confirmForgotPassword` |
| 3 | No input sanitization beyond Pydantic | **Medium** | Pydantic rejects wrong types but not malicious strings; add explicit sanitization before writing to DynamoDB string fields used in HTML rendering |
| 4 | No file content scanning | **Medium** | Add S3 event → Lambda → ClamAV pipeline; quarantine before the confirm endpoint accepts the upload |
| 5 | No audit logging for mutations | **Low** | Add an audit event DynamoDB table; log every delete, permission change, and share link operation |
| 6 | No CSRF protection | **Low** | Accepted for JWT-only flows; revisit if session cookies are ever added |
| 7 | No MFA | **Low** | Cognito advanced security is off; plan as opt-in before enterprise sales |
| 8 | Presigned URL expiry hardcoded | **Low** | Make configurable per resource sensitivity; S5 in the suggestion backlog |

Bedrock cost exposure is the highest-urgency gap: without rate limiting, a single abusive session can generate unbounded AI invocations. Add `slowapi` before enabling public sign-up.

---

## Bedrock IAM Scoping

Bedrock invocations run from the App Runner instance role. Scope the IAM policy to the minimum required model IDs:

```json
{
  "Effect": "Allow",
  "Action": "bedrock:InvokeModel",
  "Resource": [
    "arn:aws:bedrock:*::foundation-model/us.anthropic.claude-*"
  ]
}
```

Do not use `"Resource": "*"` — this allows invoking any model in the account, including expensive ones you may not have tested cost caps for.

The instance role trust policy must use `tasks.apprunner.amazonaws.com`, not `ec2.amazonaws.com`. A misconfigured trust policy causes silent 403s from Bedrock that appear as application errors, not IAM errors.

---

## Cross-References

- `core/practices/security-principles.md` — generic security principles (defense in depth, least privilege, fail-closed)
- `core/practices/anti-patterns.md` — anti-patterns including sync boto3, global state, skipping Pydantic
- `practices/infra-aws.md` — DynamoDB table patterns, S3 service patterns, Bedrock invocation syntax
- `practices/deployment-aws.md` — App Runner env var wiring, Amplify env var split, CORS JSON array format
- `practices/testing-fullstack.md` — `COGNITO_VERIFY_TOKENS=false` test flag, moto DynamoDB mocking
- `practices/conventions-fullstack.md` — `require_any_role()` usage, domain exception hierarchy

<!-- Assembled from: F-006 (architecture.md: Auth + Ownership Dependencies), F-007 (architecture.md: Cognito User Pool Configuration), F-026 (data-model.md: File Upload Limits), F-030 (debugging.md: AWS Cognito), F-052 (functionality-audit.md: 1.1 Authentication & User Management), F-054 (functionality-audit.md: 1.6 Share & Invites), F-059 (functionality-audit.md: 1.20 Account Deletion), F-061 (functionality-audit.md: 1.24 Investor Read-Only View & Auth Gate), F-063 (functionality-audit.md: 3.1 Authentication & Security Gaps), F-067 (functionality-audit.md: 4.1 Security & Compliance High Priority), F-074 (google-sso-cognito-setup.md: Step 2 Cognito hosted-UI domain), F-075 (google-sso-cognito-setup.md: Step 3 Add Google as IdP), F-076 (google-sso-cognito-setup.md: Step 4 Update App Client), F-077 (google-sso-cognito-setup.md: Step 5 Amplify env vars), F-078 (google-sso-cognito-setup.md: Step 6 Smoke test), F-079 (google-sso-cognito-setup.md: What this does NOT do), F-125 (security.md: Authentication Model), F-126 (security.md: Layer 3 Permission Service), F-127 (security.md: Layer 4 Role-based Access), F-128 (security.md: Required Secrets), F-129 (security.md: Known Security Gaps). Stack: full-stack-web-aws v0.1.0. -->
