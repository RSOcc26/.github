# 1. Multi-tenant data isolation

- **Status:** Accepted
- **Date:** 2026-10-23

## Context

Every company using CompanyCentral is a tenant, and one tenant must never see
or change another tenant's data. Data is spread over five services, each with
its own PostgreSQL database. A user can belong to several companies and
switches between them in the Hub, so the browser tells the backend which
company it is working in, and anything the browser sends can be forged.

## Options

| Option | Isolation | Operational cost |
|---|---|---|
| Database per tenant | Strongest | A database per tenant *and* per service; heavy provisioning |
| Schema per tenant | Strong | Schema creation and migrations per tenant; search-path handling per request |
| **Shared tables with `tenant_id` + row-level security** | Enforced by PostgreSQL policies | One schema, one migration per change |

## Decision

**Data ownership**

- Every service owns its **own database and credentials**. No service reads or
  writes another service's database; it uses the owner's API or events.
- Each service has a **migration role** that owns the schema and a **runtime
  role** with DML privileges only, not the table owner and without `BYPASSRLS`.

**Tenant isolation in the database**

- Tenant-owned tables are shared and have a non-null `tenant_id`, part of the
  relevant unique keys and indexes.
- **Row-level security** is enabled and forced on every tenant-owned table,
  with a policy on `current_setting('app.tenant_id')`.
- The service runs `SET LOCAL app.tenant_id = …` at the start of each
  transaction, from the verified tenant context below.
- Global data (for example the application registry) has no `tenant_id` and is
  documented as global.

**Tenant context**

1. The client *requests* a tenant (company switcher → `X-Tenant-Id`).
2. The gateway authenticates the user and checks with Identity that the user is
   an **active member** of that tenant; otherwise `403`.
3. The gateway removes client-supplied internal headers and forwards a
   **signed, short-lived tenant context** (user ID, tenant ID, roles,
   correlation ID).
4. Services verify the signature and take the tenant only from that context.
   Event consumers use the `tenantId` in the event envelope.

## Consequences

- One migration per change, independent of the number of tenants.
- Isolation depends on correct RLS policies. They are reviewed like code, and
  every service has integration tests proving that one tenant cannot read or
  change another tenant's rows and that a forged tenant header is rejected.
- Per-tenant backup and restore would need extra work.
