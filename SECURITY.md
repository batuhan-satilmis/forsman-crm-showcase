# Security

> Detailed security posture of the Forsman CRM. Written for an AppSec / GRC reader.

## OWASP Top 10 control mapping

The Forsman CRM implements explicit, testable controls against every category in the OWASP Top 10 (2021). The table below summarizes how. Each row is a real control in production, not aspirational.

| OWASP Top 10 (2021) | Risk | Forsman CRM control |
|---|---|---|
| **A01 Broken Access Control** | Users acting outside their permissions | Two-layer enforcement: API RBAC middleware + Supabase Row-Level Security policies. RLS is `FORCE`d so even direct DB access from a service role respects tenant boundaries. |
| **A02 Cryptographic Failures** | Sensitive data exposure | TLS 1.2+ everywhere (HSTS preload). Field-level AES-256-GCM encryption for PII columns; keys stored in Supabase Vault, not the application database. JWT signing key rotated quarterly. |
| **A03 Injection** | SQL, command, or template injection | Every query parameterized via Supabase JS client / `pg` parameter binding. Lint rule blocks any string-template SQL. Input validation via Zod schemas before any database call. Output encoding on all rendered fields. |
| **A04 Insecure Design** | Missing security controls in the design itself | Threat-modeled per major feature using STRIDE (see [THREAT-MODEL.md](./THREAT-MODEL.md)). Security review checklist required before merge. |
| **A05 Security Misconfiguration** | Default creds, verbose errors, open ports | No defaults shipped: every deployment requires explicit env config or fails to boot. Production error responses are sanitized — internal stack traces never reach clients. CSP, HSTS, X-Frame-Options, X-Content-Type-Options all set. |
| **A06 Vulnerable & Outdated Components** | Known-CVE dependencies | Renovate bot opens dependency PRs weekly. CI runs `npm audit --audit-level=high` and fails on high/critical. SBOM regenerated on each release. |
| **A07 Identification & Authentication Failures** | Weak auth, session hijack, credential stuffing | JWT with refresh-token rotation. Session terminated everywhere on reuse of an invalidated refresh token. Rate-limited login (5/min/IP, 10/min/user). Password reset uses single-use tokens with 15-min TTL. MFA-ready TOTP. Email enumeration blocked at the password-reset endpoint. |
| **A08 Software & Data Integrity Failures** | Tampered code, unverified webhooks | Stripe webhook signature verification on every event. Idempotency keys on every payment-creating call. CI/CD pipeline signs deployments. No dynamic `eval` anywhere in the codebase. |
| **A09 Security Logging & Monitoring Failures** | Blind to attacks | Append-only `audit_log` table for privileged actions, written in-transaction with the action itself. Application logs shipped to a separate retention path. Alerts on auth-failure spikes and refresh-token reuse. |
| **A10 Server-Side Request Forgery (SSRF)** | Server pulled into internal calls | The API does not fetch user-supplied URLs. The single endpoint that accepts an external URL (avatar import) validates against an allow-list of CDN hosts and resolves DNS server-side before fetch, blocking link-local and private IP ranges. |

## Authentication detail

### Token lifetimes & rotation

| Token | Lifetime | Where stored | Rotation |
|---|---|---|---|
| Session cookie | 7 days, sliding | HttpOnly, Secure, SameSite=Strict | Re-issued on activity |
| Access JWT | 15 min | Server-side only | Issued from session, never persisted |
| Refresh token | 30 days | HttpOnly cookie, hashed in DB | **Rotated on every refresh** |
| Password reset token | 15 min | Single-use, hashed in DB | One-shot |
| Stripe webhook signing secret | Per-environment | Env only | Annual rotation |

### Refresh-token reuse detection

If a refresh token is presented after it has already been used (i.e., the rotation path), this signals either a network replay or — more likely — a stolen token being used by an attacker while the legitimate user has already received a new token. The CRM:

1. Invalidates the entire token family (not just the offending token).
2. Force-logs-out the user on all devices.
3. Logs a `auth.refresh_reuse_detected` event to the audit log.
4. Optionally fires an alert (configurable per tenant).

This is the textbook OWASP recommendation and turns refresh-token theft into a self-detecting incident.

## Authorization detail

### Role model

```
superadmin
└── (Forsman staff only — internal admin)
tenant_owner
└── tenant_admin
    └── member
        └── viewer
```

Each role has explicit allow-lists per resource. Permissions are checked on the **action**, not the route — the same handler can be called by `tenant_admin` or `member` and produces different results based on what they're allowed to read/write.

### Tenant-isolation guarantee

> **A bug in the API authorization layer cannot leak cross-tenant data.**

This is enforced by Postgres RLS, not by hope. Every tenant-scoped table has:

```sql
CREATE POLICY tenant_isolation ON public.<table>
  USING (tenant_id = current_setting('jwt.claim.tenant_id')::uuid);
ALTER TABLE public.<table> FORCE ROW LEVEL SECURITY;
```

Tested with: an integration test that authenticates as Tenant A and attempts to read every row in every table — should see only Tenant A's rows. Run on every CI build.

## Data protection

### PII handling

- PII columns identified during data classification: `email`, `phone`, `tax_id`, `ssn_last4`, `dob`, `address`.
- Each is stored encrypted with AES-256-GCM.
- Encryption keys live in Supabase Vault; the application database has access only to ciphertext.
- A compromised DB backup cannot be decrypted without the separate key-store credentials — which are themselves rotated.

### Backups

- Managed nightly snapshots with 30-day retention.
- Point-in-time recovery within the retention window.
- Backups encrypted at rest by the provider; we add the column-level layer on top.

## Application-layer defenses

| Defense | Implementation |
|---|---|
| Input validation | Zod schemas at the boundary of every handler. Schemas are co-located with routes for review-ability. |
| Output encoding | React handles HTML escaping by default; we never use `dangerouslySetInnerHTML` on user content. |
| CSP | `default-src 'self'; script-src 'self' 'sha256-…'; …` — no `unsafe-inline`, no `unsafe-eval`. |
| CSRF | Double-submit cookie + custom header on all state-changing requests. Cookie is `SameSite=Strict`. |
| Rate limiting | Per-IP and per-user limits on auth, billing, export endpoints. Returns 429 with `Retry-After`. |
| Header hardening | HSTS preload, X-Frame-Options DENY, X-Content-Type-Options nosniff, Referrer-Policy strict-origin-when-cross-origin, Permissions-Policy minimal. |

## Payment integrity

Stripe handles PCI-relevant data — we never see a card number. But the integration itself has its own attack surface, so:

- **Webhook signature verification** on every event using the per-environment signing secret.
- **Idempotency keys** on every payment-creating call so retries are safe.
- A signed-event ledger prevents replay: each Stripe event ID is recorded on first processing; subsequent appearances are no-ops.
- Webhook handler runs in a transaction with the database write that records the event — partial state is impossible.

## Logging & monitoring

### What's logged

- All API requests: method, path, response code, duration, anonymized user ID.
- All auth events: login, logout, refresh, refresh-reuse-detected, password-reset.
- All privileged actions: admin invites, role changes, billing changes, exports.

### What's *not* logged

- PII values (we log column names changed, not the values).
- JWTs, session IDs, or any token material.
- Stripe payloads beyond event IDs and types.

### Alerting

- Spikes in 401/403 rates per IP or user.
- Refresh-token reuse events (immediate page).
- Webhook signature mismatches.
- Failed health checks.

## Vulnerability management

| | |
|---|---|
| Dependency scanning | Renovate bot + `npm audit` in CI; high/critical fail the build. |
| Static analysis | ESLint `security` ruleset + a Zod-schema lint rule that flags handlers with no input validation. |
| Dynamic testing | Burp Suite passes per major release; OWASP ZAP in CI for smoke-level coverage. |
| Threat modeling | STRIDE pass per major feature, recorded in [THREAT-MODEL.md](./THREAT-MODEL.md). |

## Compliance posture

The CRM is not formally certified, but its controls map cleanly to:

- **NIST CSF**: Identify (asset inventory, RBAC), Protect (data encryption, least privilege), Detect (audit log, alerting), Respond (runbooks, refresh-reuse detection), Recover (backups, PITR).
- **SOC 2 Common Criteria**: CC6.1 (logical access), CC6.6 (encryption), CC7.2 (anomaly monitoring).
- **HIPAA Security Rule** (where tenants handle PHI): access control, audit controls, integrity controls, transmission security, encryption.

For full audit-friendly documentation, see [ROADMAP.md](./ROADMAP.md) — formal SOC 2 readiness is on the backlog.

## Reporting a vulnerability

If you've found a security issue in *any* Forsman product, please email **security@forsmantech.com** with reproduction steps. Coordinated disclosure appreciated; we respond within 72 hours.
