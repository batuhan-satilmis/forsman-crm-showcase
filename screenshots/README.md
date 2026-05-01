# Screenshots

Place sanitized PNGs here. **Before** committing any screenshot, verify:

- [ ] No real customer names, emails, phone numbers, addresses, or company names.
- [ ] No real billing data, plan amounts, or transaction IDs.
- [ ] No real tenant IDs (UUIDs) — substitute with obvious placeholders like `tenant_demo_…`.
- [ ] No real IP addresses or geographic data.
- [ ] No URLs that include identifiers (`/tenants/abc123/...`) — blur or replace with `tenant_demo`.
- [ ] No real users in the avatar / nav header — replace with seeded demo accounts.
- [ ] No internal Stripe IDs (`cus_…`, `sub_…`).
- [ ] No internal Supabase project ref or API keys.

Suggested filenames (keep these so the README link list works):

```
01-dashboard.png            Multi-tenant dashboard with role-aware navigation
02-tenant-settings.png      Tenant settings panel showing role assignments
03-audit-log.png            Append-only audit log view
04-billing.png              Stripe-backed billing page with idempotent webhook status
05-rbac-matrix.png          (optional) Role × resource permission matrix
06-threat-model-diagram.png (optional) Architecture diagram from ARCHITECTURE.md
```

To create demo data: spin up a Forsman CRM instance against a clean tenant, seed with the names below, take screenshots, sanitize, and commit.

```
Tenant 1: Acme Demo Co.       (tenant_owner: alex@acme.demo)
Tenant 2: BetaWorks LLC       (tenant_admin: jordan@betaworks.demo)
```
