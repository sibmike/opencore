# Infrastructure Management

> See also: [`practices/plan-mode.md`](plan-mode.md) — plan-mode mechanics and exit criteria.
> See also: [`practices/feature-lifecycle.md`](feature-lifecycle.md) — where infra changes fit in the feature lifecycle.
> See also: [`practices/git-workflow.md`](git-workflow.md) — commit discipline for infra changes.
> See also: [`documentation/checklist-protocol.md`](../documentation/checklist-protocol.md) — post-deploy checklist structure.


## Infrastructure Change Approval Gate

Any net-positive infrastructure change — new database table or index, schema migration, permissions addition, infrastructure-as-code (IaC) edit, environment variable addition, managed-service config change, new compute or messaging resource — requires plan-mode approval before application. Code that depends on the new infrastructure also waits for approval.

When you realize work-in-progress will touch infrastructure, stop coding immediately:

1. Enter plan mode.
2. Write a plan listing every net infra change plus the matching docs and files that must be updated.
3. Get explicit user approval via plan-mode exit.
4. Only after approval, apply infra changes alongside dependent code in a single coherent commit.

The plan-mode gate forces the "right shape / right name / right policy?" conversation before the code that depends on the new resource is written. Missing infrastructure is the primary cause of works-locally-fails-in-production bugs.

**Exception:** Documentation-only changes that describe already-existing infrastructure do not require plan-mode approval. The rule fires on net-positive changes only.


## Net Infra Footprint Check (End of Session)

At the end of every session, compute the infrastructure delta across all managed surfaces: databases, object storage, IAM and permissions, secrets and env vars, compute, messaging, CDN and hosting config.

Document the delta explicitly in every commit message and execution plan — even when the delta is zero.

Zero-change form:

```
Net infra footprint: 0 changes.
```

Non-zero form:

```
Net infra footprint:
| Surface   | Change                              |
|-----------|-------------------------------------|
| Database  | + events table (PK/SK)              |
| IAM       | Instance role gains new permission  |
| Env vars  | + ANALYTICS_KEY (server-only)       |
```

If the delta is non-zero and the work did not go through plan-mode approval, flag it as a process violation in the session reflection.


## Post-Deploy Infrastructure Verification

Passing tests do not prove production correctness. All backend tests run with mocked or simulated cloud services. A passing test proves code logic is correct. It does not prove the cloud resource exists, has the right schema, has the right permissions, or is in the right region.

After every deploy, diff infrastructure referenced in code against what actually exists in the production environment. For each surface:

1. List all resource names referenced in source code.
2. List all resources that actually exist in production.
3. For any gap: check whether the resource is declared in IaC. If yes, deploy the template or create manually. If no, add it to IaC first, then create it.
4. Document the creation command in the commit message or execution plan so it is reproducible.
5. Verify permissions — wildcard policies may cover new resources of an existing type automatically, but new service types always need explicit policy additions.

Local tests and production verification are two distinct gates. Neither substitutes for the other.


## Setting Up a Service Does Not Mean It Is Wired

Setting up a service does not mean it is wired to all dependent services. Each integration requires explicit configuration. Dependent services often have their own separate configuration blocks that must be set. Without explicit wiring, services silently fall back to defaults.

Deliverability and integration correctness are multi-layer concerns. Each layer is independently optional but omitting any one has observable downstream costs. Document the full required configuration from day one.

Always perform a live end-to-end verification of cross-service integrations after any relevant change.

IaC tools do not automatically perform all secondary steps required to activate a service. Document manual steps explicitly in deployment guides. Do not rely on future memory.


## Deployment Sequencing

Sequence deployments to respect dependencies. The dependency must be deployed and verified before the dependent service is configured to reference it.

**DNS before SSL:** Create the hosted zone first. Update the registrar nameservers. Do not proceed to SSL certificate issuance until DNS propagation is confirmed. SSL DNS-validation challenges fail silently if authoritative NS records have not propagated.

**Email identity before auth service:** Deploy the email identity and wait for DKIM to reach verified status before deploying or configuring the auth service that references it. Deploying the auth service first causes it to fall back to a default shared sender.

**Infrastructure before code:** Apply IaC changes before deploying code that depends on them. Deploy dependent services in reverse-dependency order.

Recursively check sub-resources when planning infrastructure: check what sub-resources are needed, and sub-resources of sub-resources, all the way down.


## Environment Variables

Store environment variables in a local `.env` file that is never committed to source control.

Separate env var sets per tier (backend, frontend). Feature flags (e.g., `FEATURE_X_ENABLED=false`) let the application start without optional services enabled.

Ship `.env.example` (or equivalent) files alongside every tier that requires environment configuration. The README should reference these files explicitly and defer the full variable list to them rather than duplicating it.

For hosted front-ends that inline build-time variables: use a committed environment file as the single source of truth. Hosting-console env var overrides shadow committed files and create a split source of truth that is hard to debug. Update the committed file, commit, push — the hosting platform redeploys.

When using repository-based config for runtime services: all env vars must live in the committed config file. The hosting console's env var UI may not apply when repository config mode is active.


## CI/CD Pipeline Structure

Maintain separate CI pipelines for application code and infrastructure-as-code:

- **Application pipeline:** lint → test → deploy (triggered on push to the relevant branch).
- **Infrastructure pipeline:** validate templates → deploy (validate before apply to catch template errors early).

Maintaining separate pipelines prevents an infrastructure syntax error from blocking an application hotfix deploy.

Also maintain a separate pipeline per major tier (backend, infrastructure templates, frontend). Trigger each pipeline only on changes to its own file paths to avoid unnecessary builds. Track missing pipelines as tech debt.

For CI pipelines that deploy to cloud infrastructure:

- Prefer short-lived federated tokens over long-lived access key secrets. Create an identity provider for the CI platform, create a role trusting it scoped to the specific repository, and store only the role ARN as a repository secret.
- Scope CI roles to the minimum permissions needed for the pipeline actions.
- Auto-deploy services that watch a branch handle their own deployment. CI workflows for these services need only run lint and tests, not deployment commands.


## Local Development

Run local emulators for cloud services (a local database, a local queue) via Docker Compose so developers can work without cloud credentials. Note port conflicts when multiple services share a default port.

A Docker Compose setup should:

1. Provide a full-stack target that starts all dependencies plus the application server.
2. Provide individual service targets for running the application outside Docker while keeping service dependencies containerized.
3. Document the environment variable that switches the application from cloud endpoints to local emulator endpoints.


## Deployment Runtime Pitfalls

Always use up-to-date official documentation for any external service, library, or runtime. Do not rely on cached knowledge — APIs change, defaults change, defaults change. Verify against the current version you are deploying.

**Build-phase vs run-phase isolation:** Some runtime environments discard packages installed during the build phase before assembling the final runtime image. Install all runtime dependencies in the pre-run phase, not the build phase, when the environment separates these two phases.

**Binary availability:** Package managers may install packages without adding their CLI scripts to PATH. Use the module invocation form (e.g., `python -m <package>`) rather than bare package commands when scripts are not guaranteed to be on PATH.

**Config source mode:** When using repository-based config files for a runtime service, the hosting console's inline config UI may be disabled. All configuration must be declared in the committed config file.

**Health checks:** Use application-level health checks (HTTP against the application's health endpoint) rather than transport-level checks (TCP port open). Transport-level checks succeed as soon as the port is open, before the application is ready to serve traffic.

**VPC and outbound routing:** Routing all outbound traffic through a private network (VPC or equivalent) requires an internet egress path (NAT or equivalent) on private subnets. Without it, the container cannot reach public endpoints — auth verification, email, external APIs all fail. Deploy the egress path before or at the same time as the private-network connector.

**Startup scripts:** Use a wrapper startup script rather than a bare server command when you need to resolve secrets, run migrations, or perform other pre-flight steps. This keeps the config file simple and makes startup logic testable.

**Framework detection in SSR hosting:** Framework detection in managed hosting platforms runs at project creation time. If missed, it cannot be corrected through the UI — use the CLI. If a green build produces a placeholder page, check the detected framework first.

**Config-first for monorepos:** Use the hosting platform's monorepo configuration key (e.g., `appRoot`) to set the sub-package root. Artifact output directories are version-specific — verify the correct build output path for the version you are deploying (it changes between major versions).

**Config problems need config fixes:** If code works locally but fails in production, the problem is configuration, not code. Do not change working code based on assumptions about what a platform requires.


## Custom Domain Mapping

Use CNAME records (not A records) when the hosting service exposes a hostname rather than a static IP. A records require IP addresses; pointing them at a hostname silently fails DNS resolution.

After switching from a temporary default URL to a custom domain, update any environment variables that reference the old URL (e.g., the API base URL in the front-end) and trigger a redeploy to pick up the new value.


## Email Deliverability

Start DMARC enforcement policy at `p=none` (observation mode) for the first one to two weeks to collect aggregate reports. Escalate to `p=quarantine` only after confirming DKIM and SPF alignment in reports. Never go straight to `p=reject` without monitoring — it silently drops legitimate auth emails.

SPF guidance:

- Use `-all` (hard fail) on dedicated sending subdomains where only one service sends.
- Use `~all` (soft fail) on the apex domain if other services might send from it.

Verify deliverability with a real end-to-end test: trigger a real send and check that the email lands in Inbox, not Spam. Inspect raw headers for DKIM, SPF, and DMARC pass results. Use a third-party scorer and target a high score before wide rollout.

Email services typically start in sandbox mode (verified recipients only). Request production access early; describe only transactional sends, opt-in recipients, and bounce and complaint handling to maximize approval speed.

> See also: [`practices/security-principles.md`](security-principles.md) — credential handling and access control.


## Deployment Prerequisites Checklist

Before deploying to production, verify:

- Cloud account provisioned with appropriate access.
- Domain registered with DNS delegated to the managed DNS provider.
- SSL certificate provisioned and validated.
- Source-control connection established for auto-deploy services.
- IaC stacks deployed in dependency order.
- Email sender domain verified (DKIM status confirmed).
- Custom domain configured on each hosting service.

Maintain a separate detailed deployment doc. Use this checklist as a high-level gate only.


## Common Deployment Failure Patterns

- **Health check failures:** Check for missing env vars and port mismatches between the service config and the application's listen port.
- **CORS errors:** Verify the allowed-origins value includes the protocol prefix and correct formatting required by the runtime.
- **Auth 401/403:** Confirm credential IDs match across the identity provider config and application env vars. Verify the public key URL is reachable.
- **Email not delivering:** Check whether the email service is in sandbox mode (only verified addresses receive mail). Request production access for real users.
- **Build failures:** Verify monorepo root configuration and runtime version pinning.
- **Custom domain not resolving:** Confirm DNS records at the registrar match the provider's expected CNAME values. Allow up to 48 hours for propagation.
- **Permissions denied:** Verify resource name patterns (dashes vs underscores in resource names must exactly match what the IAM policy specifies).
- **Object storage upload failing:** Check bucket name, write permission grant, and storage CORS configuration.
- **Index-not-found errors:** Adding an index to an IaC template does not automatically apply it. Run a stack update or add the index manually.

> See also: [`practices/debugging.md`](debugging.md) — systematic root-cause investigation protocol.


## Infrastructure Doc Maintenance

Update the project infrastructure document after adding a new cloud resource, changing IaC templates, modifying CI/CD pipelines, or adding environment variables. Stale infra docs are operationally dangerous.

> See also: [`documentation/doc-maintenance.md`](../documentation/doc-maintenance.md) — cadence and ownership rules for all project docs.


<!-- Assembled from: infrastructure.md §Local Development, deployment.md §11 CI/CD Pipeline Setup, README.md §CI/CD, deployment.md §8 App Runner Setup (Console Path), deployment.md §9 Amplify Setup (UI Path), deployment.md §10 Custom Domain Configuration, deployment.md §4 Route 53 DNS Setup, deployment.md §7 SES Email Verification, README.md §Local Development with Docker, README.md §Development > Environment Variables, agent-behavior.md §10 Deployment Planning, agent-behavior.md §10.5 Infrastructure Changes Require Plan-Mode Approval, CLAUDE.md §Three Commandments / 4 Infrastructure changes require plan-mode approval, ERRATA.md §ERR-008 Cognito Verification Emails Sent from Default Shared Sender, infrastructure.md §Infrastructure (root / preamble), infrastructure.md §CI/CD Pipelines, infrastructure.md §Environment Variables, infrastructure.md §Deployment Guide, infrastructure.md §Deployment Troubleshooting, post-deploy-checklist.md §0 Net Infra Footprint Check, post-deploy-checklist.md §1 Infrastructure Verification -->
