# API Surface Platform

A defensive API security, governance, and release-readiness platform for teams that need more than a scanner.

API Surface Platform combines API discovery, validation scanning, endpoint inventory, compliance evidence, CI gate decisions, promotion controls, posture views, audit trails, and release validation into a single Go-based application.

## What it is

This repository is built for defensive monitoring and governance of your own API estate.

It is designed to support teams that need to:
- onboard API targets and specifications
- run validation scans in dev and controlled environments
- review findings, endpoint inventory, and risk summaries
- enforce CI or release gates before promotion
- monitor production posture and scheduled validation workflows
- generate HTML and PDF reports for engineering, security, and audit stakeholders

## Current platform scope

The current codebase includes:

- **Multi-workspace model**: dev, staging, and prod workspaces
- **Target and spec management**: register targets, upload specs, and version them
- **Validation scan engine**: queue-based worker flow with progress, retries, and control hooks
- **Endpoint inventory**: persisted endpoint discovery and inventory views
- **Findings workflow**: findings review, state changes, and aggregated views
- **Ownership & SLA workflow**: target owner/team fields, finding assignees, SLA due dates, overdue tracking, and MTTR/fix metrics
- **CI gate**: policy-aware gate evaluation for release decisions
- **Promotion gate**: promotion request and approval workflow
- **Scheduled scans**: recurring validation workflow for production use cases
- **Scorecard and posture**: company-level and environment-level posture views
- **Compliance mapping**: framework views and compliance evidence output
- **Audit and user management**: RBAC-aware audit trail and workspace-aware users
- **Report rendering**: HTML report plus PDF export using Chrome or Chromium, with Playwright fallback support
- **Report share links**: signed share URLs with DB-backed revoke list and optional one-time links
- **Spec diff & breaking-change gates**: spec version comparisons, breaking-change classification, and gate enforcement (CI + promotion)
- **Release validation**: smoke, release, and edge-check scripts for pre-release verification

## Architecture at a glance

The platform is organized around a few core layers:

- `cmd/` - runnable commands
- `internal/handlers/` - HTTP handlers and page/API orchestration
- `internal/repos/` - data-access layer and reporting queries
- `internal/worker/` - scan execution, validation probes, inventory, posture, and queue logic
- `internal/auth/` - auth, RBAC, workspace handling, and access resolution
- `internal/planesync/` - user and workspace projection sync across data planes
- `web/` - UI pages and front-end assets
- `templates/` - HTML and PDF templates
- `rules/` - built-in rules and taxonomy
- `policies/` - company policy examples and live policy files
- `migrations/` - schema migrations
- `scripts/ops/` - smoke, release-check, release-suite, and edge validation scripts

## Commands included

This repository currently builds multiple binaries:

- `server` - main web and API runtime
- `cigate` - CI gate command entrypoint
- `seed` - seed or bootstrap helper
- `migrate` - migration runner
- `workspace-sync` - sync users and references across data planes
- `packlint` - compliance or pack validation helper
- `ruleslint` - rules validation helper

## Quick start

### 1) Prerequisites

Recommended baseline:
- Go `1.23+`
- PostgreSQL `14+`
- Chrome or Chromium if you want PDF export from the server
- Node.js only if you want to use the optional Playwright PDF fallback tooling

### 2) Prepare environment

Create your environment file.

```bash
cp .env.example .env
set -a
source .env
set +a
```

If you do not already have `.env.example`, generate and review your `.env` from the current config model before running the server.

### 3) Build

```bash
make build
```

By default, the Makefile builds:
- `bin/server`
- `bin/cigate`
- `bin/packlint`

If you want all commands explicitly:

```bash
go build -trimpath -ldflags="-s -w" -o bin/server         ./cmd/server
go build -trimpath -ldflags="-s -w" -o bin/cigate         ./cmd/cigate
go build -trimpath -ldflags="-s -w" -o bin/seed           ./cmd/seed
go build -trimpath -ldflags="-s -w" -o bin/migrate        ./cmd/migrate
go build -trimpath -ldflags="-s -w" -o bin/workspace-sync ./cmd/workspace-sync
go build -trimpath -ldflags="-s -w" -o bin/packlint       ./cmd/packlint
go build -trimpath -ldflags="-s -w" -o bin/ruleslint      ./cmd/ruleslint
```

### 4) Run locally

```bash
make dev
```

Default endpoints:
- UI: `http://127.0.0.1:8081/login`
- Health: `http://127.0.0.1:8081/health`

## Enterprise SSO (OIDC / SSO)

The platform supports optional **OIDC-based SSO** for enterprise environments.

When enabled:
- Users authenticate via your IdP (Azure AD / Okta / Keycloak / etc.)
- Users can be **JIT provisioned** into `app_users` on first login
- Groups can be mapped to **roles** (user/auditor/admin/super_admin)
- Tenant scoping is enforced via **company projection** (users are assigned a company and remain scoped)

### Endpoints

- Start login: `GET /auth/oidc/login?next=/dashboard`
- Callback: `GET /auth/oidc/callback` (must match your IdP redirect URI)
- UI discovery: `GET /api/auth/config` (used by the login page to show one or more **Enterprise SSO** buttons)

### Environment variables

Minimal required variables to enable OIDC:

```bash
OIDC_ENABLED=true
OIDC_ISSUER_URL=https://YOUR-IDP-ISSUER
OIDC_CLIENT_ID=...
OIDC_CLIENT_SECRET=...
OIDC_REDIRECT_URL=https://YOUR-PLATFORM/auth/oidc/callback
```

Optional enterprise controls:

```bash
# UX label (login button)
OIDC_PROVIDER_LABEL=SSO

# Scopes (openid is required)
OIDC_SCOPES=openid,email,profile

# Claims
OIDC_GROUPS_CLAIM=groups
OIDC_COMPANY_CLAIM=tenant
OIDC_COMPANY_GROUP_PREFIX=company:

# JIT + sync policy
OIDC_JIT_PROVISIONING=true
OIDC_SYNC_ON_LOGIN=true
OIDC_LINK_BY_EMAIL=true
OIDC_ALLOW_CREATE_COMPANY=false

# Guardrails
OIDC_ALLOWED_DOMAINS=example.com,example.org
OIDC_REQUIRED_GROUPS=sso-users
OIDC_REQUIRE_EMAIL_VERIFIED=false

# Default mapping fallbacks
OIDC_DEFAULT_ROLE=user
OIDC_DEFAULT_COMPANY=Default

# Group → role mapping (either JSON or CSV pairs)
OIDC_GROUP_ROLE_MAP={"sso-admin":"admin","sso-auditor":"auditor"}
# or: OIDC_GROUP_ROLE_MAP=sso-admin=admin,sso-auditor=auditor

# Group → company mapping (optional)
OIDC_GROUP_COMPANY_MAP=tenant-a=CompanyA,tenant-b=CompanyB
```

### Google / Google Workspace (OIDC)

If you want a "Sign in with Google" button, you can point OIDC directly to Google:

```bash
OIDC_ENABLED=true
OIDC_PROVIDER_LABEL=Google
OIDC_ISSUER_URL=https://accounts.google.com
OIDC_CLIENT_ID=...        # Google OAuth Client ID
OIDC_CLIENT_SECRET=...    # Google OAuth Client Secret
OIDC_REDIRECT_URL=https://YOUR-PLATFORM/auth/oidc/callback

# Optional: restrict to a Workspace domain
OIDC_ALLOWED_DOMAINS=yourcompany.com
```

Notes:
- Google OIDC typically **does not include groups** in the ID token by default. If you need group-to-role mapping,
  prefer your enterprise IdP (Okta/AzureAD/Keycloak) or use SAML where groups/roles are commonly emitted.

## Enterprise SSO (SAML 2.0)

The platform also supports **SAML 2.0** (service-provider mode) for enterprise SSO.
This is especially common for Google Workspace / legacy enterprise SSO.

### Endpoints

- Start login: `GET /auth/saml/login?next=/dashboard`
- Metadata: `GET /auth/saml/metadata` (upload this to your IdP as the SP metadata)
- ACS: `POST /auth/saml/acs` (Assertion Consumer Service)
- UI discovery: `GET /api/auth/config` (login page renders "Continue with <Provider>" buttons automatically)

### Environment variables

Minimal required variables to enable SAML:

```bash
SAML_ENABLED=true

# Public URL of this platform (must match your IdP config)
SAML_ROOT_URL=https://YOUR-PLATFORM

# IdP metadata (pick one)
SAML_IDP_METADATA_URL=https://YOUR-IDP/metadata
# or: SAML_IDP_METADATA_FILE=/etc/api-surface-platform/idp-metadata.xml

# SP keypair (PEM)
SAML_SP_CERT_FILE=/etc/api-surface-platform/saml-sp.crt
SAML_SP_KEY_FILE=/etc/api-surface-platform/saml-sp.key
```

Optional enterprise controls (mirrors OIDC knobs):

```bash
# UX label (login button)
SAML_PROVIDER_LABEL=SAML

# Cookie mode used by the SAML middleware (SAML usually needs SameSite=None)
SAML_COOKIE_SAMESITE=none

# Attribute mapping (comma-separated candidates; first match wins)
SAML_EMAIL_ATTRS=email,mail,Email,http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress
SAML_NAME_ATTRS=displayName,name,cn,givenName
SAML_GROUP_ATTRS=groups,group,Groups

# Company mapping
SAML_COMPANY_ATTR=tenant
SAML_COMPANY_GROUP_PREFIX=company:
SAML_DEFAULT_COMPANY=Default

# JIT + sync policy
SAML_JIT_PROVISIONING=true
SAML_SYNC_ON_LOGIN=true
SAML_LINK_BY_EMAIL=true
SAML_ALLOW_CREATE_COMPANY=false

# Guardrails
SAML_ALLOWED_DOMAINS=example.com,example.org
SAML_REQUIRED_GROUPS=sso-users

# Group → role mapping (either JSON or CSV pairs)
SAML_GROUP_ROLE_MAP={"sso-admin":"admin","sso-auditor":"auditor"}
# or: SAML_GROUP_ROLE_MAP=sso-admin=admin,sso-auditor=auditor

# Group → company mapping (optional)
SAML_GROUP_COMPANY_MAP=tenant-a=CompanyA,tenant-b=CompanyB
```

### Database note

SSO introduces additional `app_users` columns (`auth_provider`, `oidc_issuer`, `oidc_subject`, `oidc_email_verified`, `saml_issuer`, `saml_name_id`).
Apply migrations before enabling SSO in production:

```bash
./bin/migrate up --target=all
./bin/workspace-sync --target=all --sync-users
```


## Service accounts & API tokens (PAT / machine tokens)

For enterprise deployments, integrations are expected: CI/CD, internal schedulers, and automation.
The platform provides **service accounts** + **scoped API tokens**, including **rotate / revoke**, **audit usage**, and **last used** metadata.

### UI (admin)

- **Service accounts & tokens page:** `/tokens` (admin / super_admin)

### Using a token

Supported headers:

- `Authorization: Bearer <token>`
- `X-API-Token: <token>`

Selecting data plane (workspace) for automation:

- Header: `X-Workspace: dev|staging|prod`
- Or query: `?workspace=dev|staging|prod`

### Scopes

Token scopes are expressed as strings:

- Full access: `*`
- Read-only across all API resources: `*:read`
- Resource scopes: `targets:read`, `targets:write`, `scans:read`, `scans:write`, `findings:read`, `reports:read`, etc.
- Token management endpoints require: `tokens:read` / `tokens:write`

Notes:
- If a token has an empty scope list, it is treated as **full access** unless you enable strict mode.

### Environment variables

```bash
# Enable/disable bearer tokens for /api (default: true)
API_TOKENS_ENABLED=true

# If true, tokens must have non-empty scopes (default: false)
API_TOKEN_REQUIRE_SCOPES=false

# Token usage audit mode: off | sampled | all (default: sampled)
API_TOKEN_AUDIT_MODE=sampled

# Sample interval (seconds) when mode=sampled (default: 60)
API_TOKEN_AUDIT_SAMPLE_SECONDS=60
```

### Example curl

```bash
TOKEN="asp_sa_XXXXXXXXXXXXXXXXXXXXXXXXXXXX"
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Workspace: prod" \
  http://127.0.0.1:8081/api/dashboard/summary
```



## Ownership, assignment, and SLA workflow

This platform supports a lightweight ownership and remediation workflow on top of scan findings.

### 1) Set ownership on targets

In **Scan Workspace → New Target**, you can optionally set:
- **Target owner (PIC)**: the primary person accountable for the target
- **Service owner**: owning group or on-call alias (optional)
- **Team / Squad**: the owning squad (optional)

These fields show up in target cards and executive views (Dashboard / Scorecard) to make accountability visible.

### 2) Assign findings and track due dates

Open **Result Evidence**, then on a finding click **Timeline & governance**.

In the **Ownership & SLA** panel:
- Set an **Assignee** (username) for the remediation owner
- Optionally set an **override Due date** (YYYY-MM-DD)
- Use **Clear assignee** to remove ownership
- Use **Reset due (SLA)** to remove override and fall back to SLA defaults

### 3) SLA defaults and overdue logic

SLA due dates are computed from first-seen time and severity (default policy):
- Critical: 7 days
- High: 14 days
- Medium: 30 days
- Low: 60 days
- Info/Other: 90 days

A finding is **overdue** if its effective due date is in the past while still in an active state (Open / Triaged / In Progress).

### 4) MTTR, fix rate, and reopen rate

Executive dashboards (Dashboard / Scorecard) include workflow metrics:
- **MTTR**: mean time to remediate (from first seen → resolved)
- **Fix rate**: resolved / opened within the window
- **Reopen rate**: reopened after a fixed/verified/closed resolution

These metrics are windowed (30d and 90d where applicable) and scoped by company (and workspace access).
- Readiness: `http://127.0.0.1:8081/readyz`

## Database and workspace model

The platform is designed around a core database plus environment data planes.

Typical setup:
- `core` database for shared identity and control data
- `dev` database for development validation and day-to-day scanning
- `staging` database for pre-release evaluation
- `prod` database for production posture, monitoring-oriented flows, and scheduled validation

After a fresh setup, run migrations and sync workspace references:

```bash
./bin/migrate up --target=all
./bin/workspace-sync --target=all --sync-users
```

## Report share links

The server can generate signed, read-only share links for reports (useful for auditors, management, and external reviewers).

- Links are signed using `REPORT_SHARE_SECRET`
- DB-backed token records support **revoke** and optional **one-time** links
- A share link opens the report via `/share/report?token=...`

Required environment variables:

```bash
REPORT_SHARE_SECRET=...random...
REPORT_SHARE_DEFAULT_TTL_MINUTES=10080   # 7d default
REPORT_SHARE_MAX_TTL_MINUTES=43200       # 30d max
```

## Spec version diff and breaking-change gates

The platform supports spec version diffing and a breaking-change analyzer between spec versions. Results are stored per scan and can be enforced by CI and promotion gates.

Breaking change classes (best-effort):
- `removed_path`
- `removed_operation`
- `request_schema_tightening`
- `response_schema_breaking`
- `auth_requirement_change`

Company policy keys:

```yaml
spec:
  blockIf:
    breakingChanges: true
    maxBreakingChanges: 0   # 0 = fail on any breaking change
```

Relevant API routes:
- `GET /api/scans/{scanId}/spec-diff` (scan-scoped diff summary, auto-backfilled if missing)
- `GET /api/specs/{specId}/diff?baseSpecId=...` (spec-to-spec diff)

## Ticketing integration (Jira + generic webhook)

The platform can create external tickets from:
- a **finding** (manual button / API call)
- a **CI gate failure** (optional auto-create)
- a **promotion rejection** (optional auto-create)

Every ticket is stored as an `integration_issue` record (provider, external ID, status, URL) and can be synced read-only.

### Configuration

Core env keys:

```bash
TICKETING_PROVIDER=auto            # auto|jira|webhook
TICKETING_TIMEOUT_SEC=8

# Auto-create tickets (optional)
TICKETING_AUTO_CREATE_CI_FAIL=false
TICKETING_AUTO_CREATE_PROMOTION_REJECT=false

# Dedupe behavior
TICKETING_DEDUPE_PROMOTION_TO_CI=true   # promotion rejection reuses CI ticket for the same scan
TICKETING_COMMENT_ON_REUSE=true         # add a small comment when a ticket is reused

# Read-only status sync (optional background loop)
TICKETING_AUTO_SYNC_ENABLED=false
TICKETING_AUTO_SYNC_INTERVAL_SEC=600
TICKETING_AUTO_SYNC_MIN_AGE_SEC=300
TICKETING_AUTO_SYNC_LIMIT=50

# Priority mapping (best-effort; Jira names vary by instance)
TICKETING_PRIORITY_CRITICAL=Highest
TICKETING_PRIORITY_HIGH=High
TICKETING_PRIORITY_MEDIUM=Medium
TICKETING_PRIORITY_LOW=Low
TICKETING_PRIORITY_INFO=Lowest
```

#### Jira

```bash
TICKETING_JIRA_BASE_URL=https://your-domain.atlassian.net
TICKETING_JIRA_EMAIL=you@company.com
TICKETING_JIRA_API_TOKEN=...token...
TICKETING_JIRA_PROJECT_KEY=SEC
TICKETING_JIRA_ISSUE_TYPE=Task
TICKETING_JIRA_LABELS=api-surface-platform,ci-gate
```

#### Generic ticket webhook

```bash
TICKETING_WEBHOOK_URL=https://ticketing-gateway.internal/hooks/api-surface
TICKETING_WEBHOOK_BEARER_TOKEN=...optional...
```

Webhook contract (best-effort):
- Create: POST JSON with `action=create_ticket`
- Sync: POST JSON with `action=sync_ticket`

The webhook response should return any of:
- `externalId` (or `id`/`key`)
- `url`
- `status`

### API routes

- Create from finding:
  - `POST /api/findings/{findingId}/ticket`
  - Body: `{ "provider":"auto", "title":"...", "note":"...", "labels":["..."] }`

- List tickets (finding-scoped):
  - `GET /api/findings/{findingId}/tickets`

- List integration issues (company-scoped):
  - `GET /api/integrations/issues?limit=100`
  - Optional filter: `resourceType` + `resourceId`

- Sync ticket status (read-only):
  - `POST /api/integrations/issues/{id}/sync`

If `TICKETING_AUTO_SYNC_ENABLED=true`, the server will also refresh open issues in the background at the configured interval.


## PDF rendering

### Recommended
Use Chrome or Chromium available on the host.

### Optional fallback
Install Playwright tooling only if you need it:

```bash
make pdf-deps
make pdf-browsers
```

The fallback renderer script is located at:
- `scripts/pdf/render_playwright.mjs`

## Release validation workflow

The repository now includes a release-validation toolchain under `scripts/ops/`.

Main scripts:
- `smoke.sh` - public and basic smoke checks
- `smoke-auth.sh` - authenticated smoke checks
- `release-check.sh` - single release-check run with artifacts
- `edge-check.sh` - edge-case validation
- `release-suite.sh` - full release suite runner for admin, user, and reviewer validation

Important behavior:
- the release suite is **workspace-aware**
- target- or scan-scoped checks are discovered from the active workspace when possible
- if a workspace such as `prod` is posture-oriented and legitimately has no manual target or scan, the suite now **warns/skips** those checks instead of producing false failures
- reviewer-only routes are validated according to the role model

Example release-suite run:

```bash
BASE_URL=http://127.0.0.1:8081 \
ADMIN_USERNAME='admin' \
ADMIN_PASSWORD='your-password' \
ADMIN_WORKSPACE='prod' \
USER_USERNAME='user' \
USER_PASSWORD='user-password' \
USER_WORKSPACE='prod' \
AUDITOR_USERNAME='auditor' \
AUDITOR_PASSWORD='auditor-password' \
AUDITOR_WORKSPACE='prod' \
FRAMEWORK='pci' \
bash scripts/ops/release-suite.sh
```

The suite writes logs, summaries, and archives under:
- `scripts/ops/out/`


## Notification workflow integration

The platform now includes a webhook-first notification backbone for workflow events.

Supported outbound targets:
- generic webhook
- Slack incoming webhook
- Microsoft Teams incoming webhook

Current workflow events:
- `scan.completed`
- `scan.failed`
- `scheduled_scan.failed`
- `ci_gate.failed`
- `promotion.decision`
- `posture.threshold_breached`
- `accepted_risk.expiring`
- `accepted_risk.expired`

Notification settings are environment-driven and can also be overridden through `app_settings` runtime overrides.

Main notification env vars:
- `NOTIFY_WEBHOOK_URL`
- `NOTIFY_SLACK_WEBHOOK_URL`
- `NOTIFY_TEAMS_WEBHOOK_URL`
- `NOTIFY_TIMEOUT_SEC`
- `NOTIFY_SCAN_COMPLETE_ENABLED`
- `NOTIFY_SCAN_FAILED_ENABLED`
- `NOTIFY_CI_GATE_FAIL_ENABLED`
- `NOTIFY_PROMOTION_DECISION_ENABLED`
- `NOTIFY_SCHEDULED_SCAN_FAIL_ENABLED`
- `NOTIFY_POSTURE_THRESHOLD_ENABLED`
- `NOTIFY_POSTURE_THRESHOLD_BAND`
- `NOTIFY_ACCEPTED_RISK_EXPIRY_ENABLED`
- `NOTIFY_ACCEPTED_RISK_EXPIRY_LEAD_DAYS`
- `NOTIFY_ACCEPTED_RISK_DEFAULT_EXPIRY_DAYS`

Accepted-risk lifecycle notes:
- moving a finding into `accepted_risk` now persists a governed exception record with:
  - business reason
  - required compensating control
  - review due date
  - expiry date
  - reminder lead days
  - approving actor (`accepted_by` username / role)
  - leaving `accepted_risk` closes the active exception lifecycle record and returns the finding to normal remediation workflow
  - the worker sweeps active accepted-risk records and emits `accepted_risk.expiring` and `accepted_risk.expired` without repeating the same reminder on every tick
  - lifecycle history is append-only and persisted in `finding_lifecycle_events`, so Timeline cards survive page refresh and remain visible to auditors
  - webhook URLs are treated as secrets and are not returned in the runtime settings snapshot
  - Production Posture includes an accepted-risk register panel and an API endpoint at `/api/posture/accepted-risks` so exception debt stays visible during review
  - the accepted-risk register now supports enterprise review actions:
  - open latest evidence
  - open finding timeline directly from the latest result page
  - renew / extend an exception with fresh governance dates
  - return an exception to remediation
  - report HTML / PDF include an **Exception Register** section with:
  - active / expiring / expired / review due counts
  - exception debt badge
  - average / max aging
  - sample top exceptions with owner and control note

## Configuration highlights

Common environment variables include:
- `APP_ENV`
- `APP_ADDR`
- `CORE_DATABASE_URL`
- `DEV_DATABASE_URL`
- `STAGING_DATABASE_URL`
- `PROD_DATABASE_URL`
- `WORKSPACE_ENV`
- `ACTIVE_DATA_PLANE`
- `RULES_PATH`
- `RULES_FALLBACK_PATH`
- `POLICY_DIR`
- `COMPLIANCE_PACK_DIR`
- `CSRF_ALLOWED_ORIGINS`
- `CSRF_REQUIRE_ORIGIN`
- `SESSION_TTL_HOURS`
- `SESSION_IDLE_TIMEOUT_MINUTES`
- `COOKIE_SECURE`
- `COOKIE_SAMESITE`
- `DB_ERROR_TRACE`
- `MONITOR_PRODUCTION_METRICS_URL`
- `ALLOW_UI_RESTART`

Use stricter settings for staging and production than for local development.

## Governance data model additions

Recent migrations added the governance and timeline persistence tables below:
- `0010_finding_risk_acceptances.sql` -> active accepted-risk lifecycle records
- `0011_finding_exception_governance.sql` -> review due date, status, and governance indexes
- `0012_finding_lifecycle_events.sql` -> append-only lifecycle timeline for finding actions

Apply these migrations before testing accepted-risk governance, posture register actions, or persisted Timeline cards.

## UI modules

The web layer currently includes pages for:
- dashboard
- scan and result review
- scorecard
- posture
- monitor
- scheduled scans
- CI gate
- promotion gate
- audit
- users
- settings

## Role and reviewer boundary

The intended platform model is **maker-checker**:
- `user` -> operate scans, review posture, fix blockers
- `auditor` -> reviewer and approval role
- `admin` -> operational reviewer and release authority
- `super_admin` -> highest-trust reviewer and platform operator

Promotion Gate is a **review board**. It should only be visible to reviewer roles:
- `auditor`
- `admin`
- `super_admin`

User-facing release reasoning belongs in:
- Result
- CI Gate
- Posture

Those pages should explain blockers and promotion status without exposing reviewer-only approval controls.

## DAST operating model

The platform now supports policy-driven DAST execution guardrails.

Current DAST modes:
- `observe` - posture-first, low-risk, production-friendly
- `safe` - default active validation posture for dev/staging
- `controlled` - explicitly approved non-production posture for stricter validation

Recommended environment approach:
- `dev` -> `safe`
- `staging` -> `safe`
- `production` -> `observe`

See also:
- `docs/ROLE_AND_RELEASE_MODEL.md`
- `docs/DAST_POLICY_AND_RUNBOOK.md`
- `policies/_example.company-policy.yaml`
- `policies/_example.company-policy.prod-observe.yaml`


## Security and operational notes

- This project is intended for defensive monitoring and governance of systems you own or are authorized to assess.
- Keep `.env` and database credentials out of Git.
- Prefer running behind nginx or another reverse proxy in shared environments.
- Validate `/health` and `/readyz` before smoke or release-suite execution.
- Run `release-suite.sh` after major changes and before promoting a build to a shared dev server.

Session hardening:
- `SESSION_TTL_HOURS` controls overall cookie/session expiry.
- `SESSION_IDLE_TIMEOUT_MINUTES` expires sessions after inactivity (best-effort). The UI will auto-logout on idle and the server also enforces inactivity expiry when enabled.

Mandatory 2FA (Google Authenticator / TOTP):
- `MFA_TOTP_REQUIRED=true` enforces TOTP for **local** logins. Users without 2FA will be forced to enroll during login.
- `MFA_TOTP_ENC_KEY` is required when `MFA_TOTP_REQUIRED=true`. It must be **base64** and decode to **32 bytes** (AES-256-GCM).
  - generate: `openssl rand -base64 32`
- `MFA_TOTP_ISSUER` controls the issuer label shown in authenticator apps (default: `API Security Platform`).
- `MFA_TOTP_SKEW_STEPS` allowed time skew steps (0–3, default 1). Each step is 30 seconds.
- `MFA_TOTP_REQUIRE_FOR_SSO=true` (optional) enables **step-up MFA after SSO** (OIDC/SAML). If enabled, users will be redirected to the login MFA panel after SSO and must enter a TOTP code before a session is issued.

Admin ops:
- Reset user TOTP (e.g., user changed phone): `POST /api/users/{id}/mfa/reset` (admin/super_admin).

Troubleshooting:
- If `/api/login` returns `{ ok:false, mfa:{ stage:"enroll" ... } }`, that is **expected** for first-time enrollment.
  Add the secret to Google Authenticator, then submit the 6-digit code to `POST /api/mfa/verify` with `challengeId`.
- If you see `code already used`, wait for the next 30-second code and try again.

## Repository status

The current platform state includes:
- split report HTML and PDF handling for easier maintenance
- hardened workspace validation
- hardened spec validation and oversize request handling
- user projection sync across data planes
- release-suite, smoke, and edge validation scripts
- admin and user release validation flow

## Development workflow

Useful commands:

```bash
make fmt
make lint
make test
make build
make dev
```

## License

Add the license file or policy that matches your deployment model before publishing outside a private repository.
