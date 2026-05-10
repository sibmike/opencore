# Security Principles

> See also: [practices/anti-patterns.md](anti-patterns.md) | [practices/infra-management.md](infra-management.md) | [practices/architecture-principles.md](architecture-principles.md)

---

## Account & Infrastructure Hardening

Secure the cloud root account immediately after creation:

1. Enable MFA on the root account via the provider's identity dashboard.
2. Create a dedicated admin user for day-to-day work. Never use the root account operationally. Enable MFA on the admin user and attach full-admin permissions.
3. Lock the target region in every console session before creating resources. Mixing regions silently scatters infrastructure and breaks cross-stack references.
4. Before starting deployment, verify that every managed service you depend on is available in the chosen region. Access gates vary by region and account tier.

---

## TLS Certificates

When provisioning TLS certificates via a cloud certificate manager:

1. Verify the certificate's region matches the region of every service that consumes it. Some edge services require a specific region (e.g., a global CDN may require `us-east-1`) regardless of where the application runs.
2. Use DNS validation, not email validation. DNS validation auto-renews without inbox dependency.
3. Request the apex domain and a wildcard (`*.example.com`) in a single certificate to cover all subdomains.
4. Allow 5–30 minutes for issuance after DNS records propagate.

---

## Authentication

Use an OIDC-compliant identity provider. Never build a bespoke authentication system.

**JWT flow:**

1. The user authenticates via the identity provider (hosted UI or custom form).
2. The auth library stores access, ID, and refresh tokens in browser storage.
3. The API client reads the access token from the auth session.
4. Every request sends `Authorization: Bearer {token}`.
5. The backend verifies the token signature against the provider's JWKS endpoint.
6. The auth middleware extracts the user identifier from the verified token and passes it to downstream handlers.

**Backend rules:**
- Production always verifies against the live JWKS endpoint. Never skip verification in production.
- Tests may use a flag to bypass verification. That flag must be unreachable in production builds.
- Fail closed: if token verification fails for any reason, return 401 or 403. Never fall back to unauthenticated access.

---

## Secrets & Access Control

**Secrets management:**
- Never commit secrets files (e.g., `.env`). Enforce exclusion via `.gitignore`.
- Store all secrets as environment variables. Never hardcode API keys, passwords, or tokens in source.
- Store CI/CD secrets in the CI platform's secrets store. Inject at build time. Never log them.
- No secrets in source code, comments, or commit messages.

**Least privilege:**
- Scope every permission role to the specific resources it needs. No wildcard permissions.
- Audit and tighten permissions after initial deployment. Default-open is an anti-pattern.
- Service identities get only the permissions their runtime code actually calls.

> See also: [practices/infra-management.md](infra-management.md) for the plan-mode approval requirement on IAM additions.

---

## Core Principles

**Validate at boundaries.** Validate all input via schema or type models at the API entry point. Interior code trusts validated types and does not re-validate.

**Defense in depth.** Enforce critical constraints (file size limits, rate limits, access checks) at multiple layers: client, server, and storage. No single layer is the sole guard.

**Fail closed.** If authentication or authorization fails, reject the request. No fallback to a less-privileged or unauthenticated path.

**Credential hashing.** Hash shared secrets (passwords, passcodes) with a strong algorithm and per-secret salts. Never store or compare credentials in plain text.

**Minimize blast radius.** Add security-sensitive infrastructure (new tables, indices, IAM roles) as isolated, single-purpose resources. Attaching security-sensitive state to an existing high-traffic resource increases the blast radius of a misconfiguration.

> See also: [practices/anti-patterns.md](anti-patterns.md) for common security anti-patterns to avoid.

<!-- Assembled from: deployment.md §3. AWS Account Setup, deployment.md §5. SSL Certificate (ACM), security.md §Authentication Model, security.md §Secrets Management, security.md §Security Principles -->
