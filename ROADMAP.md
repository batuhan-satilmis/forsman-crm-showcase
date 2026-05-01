# Roadmap

> Hardening backlog and feature plans for the Forsman CRM. Public roadmap, prioritized.

## Now (in flight)

- [ ] **WebAuthn / passkeys** as a TOTP alternative for tenants who want phishing-resistant auth.
- [ ] **Tenant-level audit-log export** (signed PDF + CSV) for SOC 2 / HIPAA evidence requests.
- [ ] **Per-tenant data-residency selector** (US-East default, EU available) — schema is region-aware; needs UX.

## Next (next quarter)

- [ ] **SOC 2 Type 1 readiness**: formalize the controls already in place into auditor-friendly documentation. Map [SECURITY.md](./SECURITY.md) controls to TSC criteria.
- [ ] **SSO via SAML / OIDC** for tenants on the enterprise tier.
- [ ] **Automated DR drill** quarterly: restore a backup to a sandbox and validate the column-level encryption keys.
- [ ] **Bug bounty program** (private) with a coordinated-disclosure policy.

## Later (research / scoping)

- [ ] **Just-in-time admin access**: tenant_owner grants tenant_admin a time-boxed elevated role for a specific maintenance window.
- [ ] **Real-time DLP** on the export endpoint to catch accidental PII spills.
- [ ] **Tenant-level rate-limit overrides** for high-volume integrations.

## Done

- [x] OWASP Top 10 baseline coverage (see [SECURITY.md](./SECURITY.md)).
- [x] Multi-tenant isolation with API + RLS double-enforcement.
- [x] Refresh-token rotation with reuse-detection.
- [x] Append-only audit log with FORCE RLS.
- [x] Stripe webhook signature verification + idempotent processing.
- [x] STRIDE threat-model first pass (see [THREAT-MODEL.md](./THREAT-MODEL.md)).
- [x] Backend hardening pass 1 (Q4 2025).
- [x] Frontend hardening pass 2 (Q1 2026).
- [x] Payment-integrity hardening pass (Q1 2026).
