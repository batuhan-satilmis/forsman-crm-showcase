# Forsman CRM — Architecture & Security Showcase

> A production-deployed, multi-tenant SaaS CRM I designed, built, and continue to harden as sole engineer at Forsman Technology & Consulting. **This repository is a public architectural overview — the source code remains proprietary.**

---

## Why this repo exists

The Forsman CRM is the flagship product of [Forsman Technology & Consulting](https://forsmantech.com). Because it serves real client tenants and revenue, the source code stays private. But the **engineering and security work behind it** is the centerpiece of my portfolio — and recruiters, hiring managers, and security-minded buyers deserve a way to evaluate it.

This repo walks through the system architecture, the security model, the threat coverage, and the engineering decisions, with **redacted snippets and diagrams only**. There is no runnable code here.

---

## TL;DR

- **What**: Multi-tenant B2B SaaS CRM with embedded billing, tenant-isolated data, role-based access control, and audit logging.
- **Stack**: React + Vite + Tailwind on Vercel · Node.js + Express on Railway · Supabase / PostgreSQL · Stripe for payments.
- **Security posture**: Full **OWASP Top 10** control set implemented and re-tested in iterative hardening passes. JWT with refresh-token rotation, Supabase **Row-Level Security** for tenant isolation, field-level PII encryption, server-side input validation, CSP & CSRF defenses, Stripe webhook signature verification with replay protection.
- **My role**: Sole architect, developer, and security owner. Production deployment, ongoing multi-pass threat-modeling and hardening.

---

## Screenshots

> Screenshots are stored under [`/screenshots`](./screenshots/) and reference UI states, not data. Real client information is never shown.

| | |
|---|---|
| ![Dashboard](./screenshots/01-dashboard.png) | ![Tenant settings](./screenshots/02-tenant-settings.png) |
| Multi-tenant dashboard with role-aware navigation | Tenant settings panel with RBAC enforcement |
| ![Audit log](./screenshots/03-audit-log.png) | ![Stripe billing](./screenshots/04-billing.png) |
| Privileged-action audit log | Stripe-backed billing with idempotent webhook handlers |

*(Add real screenshots after sanitizing — see [`/screenshots/README.md`](./screenshots/README.md).)*

---

## Documents in this repo

- 📐 [**ARCHITECTURE.md**](./ARCHITECTURE.md) — System architecture, deployment topology, data flow.
- 🛡️ [**SECURITY.md**](./SECURITY.md) — OWASP Top 10 control mapping, security controls in detail.
- 🎯 [**THREAT-MODEL.md**](./THREAT-MODEL.md) — STRIDE-based threat model with attack trees and mitigations.
- 🗺️ [**ROADMAP.md**](./ROADMAP.md) — Hardening backlog and planned features.

---

## Highlights at a glance

### Authentication & session management
- JWT access tokens with **rotating refresh tokens**, secure HTTP-only cookies, and server-side revocation on logout/compromise.
- MFA-ready flow (TOTP) and rate-limited password reset with email-enumeration protection.

### Authorization & multi-tenancy
- **Two-layer enforcement**: API middleware (RBAC) + database (Supabase Row-Level Security policies). A bug at the API layer alone cannot leak cross-tenant data — RLS catches it.
- Roles: `superadmin`, `tenant_owner`, `tenant_admin`, `member`, `viewer`. Permissions enforced per-resource, per-action.

### Data protection
- **Field-level encryption** (AES-256-GCM) for PII columns, with key isolation outside the application database.
- Least-privilege Postgres service roles. Database connections never use the same credentials as application reads.
- **Audit log** for every privileged action (admin invites, role changes, billing events, exports), tamper-evident via append-only design.

### Application defenses
- Server-side input validation on every endpoint (Zod schemas).
- Parameterized queries everywhere (no string-concatenated SQL anywhere in the codebase).
- CSRF tokens on all state-changing routes; SameSite=Strict on session cookies.
- XSS mitigation: output encoding by default plus a strict Content-Security-Policy.
- API rate limiting (per-user and per-IP) on auth, billing, and export endpoints.

### Payment integrity
- Stripe **webhook signature verification** on every event.
- **Idempotency keys** on every payment-creating call to prevent double-charges from retries.
- Replay protection via signed timestamp + server-side processed-event ledger.

### Process
- Threat modeling done **iteratively**, not once. Each major feature ships with a STRIDE pass and an entry in [THREAT-MODEL.md](./THREAT-MODEL.md).
- Multi-pass hardening cycles post-deployment. The current state is the result of three explicit hardening passes (frontend, backend, payment integrity).

---

## What I'd want a hiring manager to look at first

1. **[SECURITY.md → OWASP Top 10 mapping](./SECURITY.md#owasp-top-10-control-mapping)** — proves I can map theory to working controls.
2. **[ARCHITECTURE.md → multi-tenant isolation](./ARCHITECTURE.md#tenant-isolation)** — proves I can reason about the boundary that matters most in B2B SaaS.
3. **[THREAT-MODEL.md → payment-flow attack tree](./THREAT-MODEL.md#payment-flow)** — proves I think like an attacker, not just a builder.

---

## Contact

- 💼 [LinkedIn](https://linkedin.com/in/batuhan-satilmis)
- 🌐 [forsmantech.com](https://forsmantech.com)
- ✉️ batuhan@satilmis.me

---

## License & disclosure

The contents of this repository (architectural diagrams, written analysis, redacted snippets) are © 2026 Forsman Technology & Consulting LLC, made available for portfolio review under [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/). The Forsman CRM source code, schema, secrets, infrastructure configuration, and customer data are not included and are not licensed for any use.
