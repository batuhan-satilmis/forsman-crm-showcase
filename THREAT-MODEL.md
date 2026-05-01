# Threat Model

> STRIDE-based threat model for the Forsman CRM. Updated iteratively, not just at design time. Each threat below is paired with the implemented mitigation.

## Methodology

- **Framework**: STRIDE (Spoofing · Tampering · Repudiation · Information disclosure · Denial of service · Elevation of privilege).
- **Cadence**: full pass at design time per major feature; lightweight re-pass during code review; full re-pass quarterly.
- **Output**: this document, plus per-feature entries in the engineering wiki (private).
- **Companion mapping**: where applicable, threats are tied to MITRE ATT&CK techniques.

---

## Trust boundaries

```
┌─────────────────┐    TB1     ┌─────────────────┐
│  Internet user  │ ─────────► │ Vercel CDN/edge │
└─────────────────┘            └─────────────────┘
                                       │ TB2
                                       ▼
                               ┌─────────────────┐
                               │  Express API    │
                               │  (Railway)      │
                               └─────────────────┘
                                       │ TB3
                                       ▼
                               ┌─────────────────┐
                               │  Supabase /     │
                               │  PostgreSQL     │
                               └─────────────────┘

External callers:
  - Stripe   ──signed events──► API   (TB4)
  - SES/email └── outgoing only ─┘
```

| Boundary | What crosses | Trust assumption |
|---|---|---|
| TB1 | Public HTTPS traffic | None — assume hostile. |
| TB2 | Authenticated API calls | Session cookie trusted only after JWT verify; tenant claim trusted only against RLS. |
| TB3 | DB queries | DB is trusted-but-verified — RLS enforces what the API layer claims. |
| TB4 | Stripe webhook events | Trusted only after signature verification + replay-ledger check. |

---

## STRIDE walkthrough

### Spoofing

| # | Threat | Mitigation |
|---|---|---|
| S-1 | Attacker logs in as another user via stolen password. | Rate-limited login (5/min/IP), MFA-ready TOTP, alerting on auth-failure spikes. Refresh-token rotation limits the window of credential value. |
| S-2 | Attacker forges a session by guessing a session ID. | Session IDs are 256-bit random opaque values stored server-side. No client-derivable identity. |
| S-3 | Attacker forges a Stripe webhook to credit their account. | Every webhook signature-verified with `Stripe-Signature` header against the per-environment signing secret. Failed verification → 401, no DB write. |
| S-4 | Attacker forges a JWT to claim admin role. | JWTs signed with HS256 against a per-environment secret in Supabase Vault. No `alg: none` accepted. Tenant claim is checked against RLS — even a forged JWT can't read another tenant's data. |

### Tampering

| # | Threat | Mitigation |
|---|---|---|
| T-1 | Attacker modifies request body to change `tenant_id` and read another tenant's data. | API resolver pulls `tenant_id` from the JWT, never from the body or path. Mismatch → 403. RLS blocks at the DB layer regardless. |
| T-2 | Attacker tampers with audit log to hide actions. | Audit log is append-only at the application layer (no UPDATE/DELETE handlers). Postgres role used by the API has no `UPDATE` or `DELETE` permission on `audit_log`. |
| T-3 | Attacker tampers with refresh token to extend session. | Refresh tokens are random 256-bit values stored hashed in the DB; their "value" is opaque. Tampering produces a hash that doesn't match anything. |

### Repudiation

| # | Threat | Mitigation |
|---|---|---|
| R-1 | A tenant admin claims they didn't change a billing setting. | Audit log records actor, action, target, before/after, timestamp, IP, user-agent. Tied to a specific JWT/session. |
| R-2 | An employee/intern claims they didn't issue a refund. | Same audit log. Plus Stripe's own ledger — independent third-party log. |

### Information disclosure

| # | Threat | Mitigation |
|---|---|---|
| I-1 | Database backup is leaked or stolen. | PII is field-level AES-256-GCM-encrypted; keys live outside the DB in Supabase Vault. A backup alone is not enough to decrypt PII. |
| I-2 | Cross-tenant read via API bug. | Three-layer enforcement (JWT claim, API RBAC, DB RLS). RLS is `FORCE`d so a forgotten `WHERE tenant_id = ...` does not leak. |
| I-3 | XSS leaks session cookie. | Session cookie is `HttpOnly` — JavaScript cannot read it even if XSS lands. CSP blocks inline script execution. |
| I-4 | Verbose error messages leak schema or paths. | Production responses are sanitized: only an error code and a request ID. Detailed errors logged server-side under the same request ID. |
| I-5 | Email-enumeration attack on /password-reset. | Endpoint always returns the same response regardless of whether the email exists. Rate-limited. |

### Denial of service

| # | Threat | Mitigation |
|---|---|---|
| D-1 | Brute-force login lockout (auth fatigue). | Per-IP and per-user rate limits with `Retry-After`. Backoff is exponential. Account itself is *not* locked, to prevent attacker-induced denial of legitimate user. |
| D-2 | Expensive query exhausts DB. | All list endpoints paginated, with maximum page size enforced server-side. No `SELECT *` from clients. |
| D-3 | Unsigned-request flood at the edge. | Vercel + Railway absorb anonymous load; rate limits kick in at the API layer. |

### Elevation of privilege

| # | Threat | Mitigation |
|---|---|---|
| E-1 | `member` calls an admin-only endpoint. | RBAC middleware (`requireRole(['tenant_admin', 'tenant_owner'])`) returns 403 before the handler runs. |
| E-2 | `member` invokes the admin endpoint via a tampered role claim. | Role is read from server-issued JWT, not from the request body. |
| E-3 | Privilege escalation via SQL injection. | Parameterized queries everywhere; lint rule blocks string-templated SQL; even if injection landed, the limited Postgres role used by the API cannot grant roles or write to `audit_log`. |

---

## Payment flow attack tree

A specific area worth zooming into. Stripe handles the card data; **we** handle the order of operations, and that's where most payment exploits actually live.

```
Goal: get a paid subscription without paying
├── Replay a successful Stripe webhook event ID
│   └── Mitigation: signed-event ledger; idempotent processing
├── Inject a forged webhook event
│   └── Mitigation: Stripe-Signature verification with per-env secret
├── Race two concurrent /subscribe requests so we get billed once but provisioned twice
│   └── Mitigation: idempotency keys, transactional state transitions, unique constraint on (tenant_id, stripe_subscription_id)
├── Cancel via UI but cancellation never reaches Stripe (we still bill them — bad for business, also a trust issue)
│   └── Mitigation: cancellation only applied locally after Stripe ack; UI optimistic updates rolled back on failure
├── Modify amount in client request before /subscribe
│   └── Mitigation: amount is server-derived from plan_id; client cannot supply amount
└── Use stolen card to buy seats, then dispute
    └── Mitigation: Stripe Radar handles fraud; we alert on chargebacks and freeze tenant exports until resolved
```

---

## Re-modeling cadence

Per major feature: file a doc following [`/templates/stride-worksheet.md`](https://github.com/batuhan-satilmis/threat-modeling-framework/blob/main/templates/stride-worksheet.md) (in the [threat-modeling-framework](https://github.com/batuhan-satilmis/threat-modeling-framework) repo) → review with senior engineer or security peer → entries added here.

Quarterly: full top-to-bottom STRIDE re-pass. Past passes:

- **2025-Q4** (initial): focus on auth, multi-tenancy, billing.
- **2026-Q1** (post-launch hardening): focus on logging, alerting, abuse paths.
- **Next**: Q2 — focus on data export paths and tenant deletion / data-retention flows.

---

## Out of scope (for this doc)

- Physical security of provider data centers (delegated to AWS, Stripe, Supabase, Railway, Vercel).
- Insider threats from Forsman staff (covered by a separate access-control SOP, not public).
- Supply-chain attacks on dependencies (covered by the [vulnerability-management process in SECURITY.md](./SECURITY.md#vulnerability-management)).
