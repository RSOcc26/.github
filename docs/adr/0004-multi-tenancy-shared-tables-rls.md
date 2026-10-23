# 4. Multi-tenancy with shared tables, `tenant_id` and row-level security

- **Status:** Accepted. Supersedes the earlier schema-per-tenant proposal.
- **Date:** 2026-10-23

## Context

The CompanyHub draft proposed one PostgreSQL schema per tenant in every
service database. With five services that means creating and migrating
schemas for every new company, connection or search-path handling per request,
and migrations whose runtime grows with the number of tenants. That is a lot
of operational work for a two-person team and a small number of tenants.

## Decision

- Tenant-owned tables are **shared** and have a non-null `tenant_id` column
  that is part of every unique key and index where relevant.
- **Row-level security** is enabled (and forced) on every tenant-owned table, with a
  policy `tenant_id = current_setting('app.tenant_id')::uuid`.
- The service sets `SET LOCAL app.tenant_id = …` at the start of every
  transaction, from the **verified** tenant context (see ADR-0008), never from
  request input.
- The runtime database role does not own the tables and has no `BYPASSRLS`.
  Cross-tenant maintenance runs under the migration role, explicitly.
- Global data (for example the application registry in Access) has no
  `tenant_id` and is documented as global.
- Every service has integration tests proving that one tenant cannot read or
  change another tenant's rows.

## Consequences

- One migration per change, regardless of the number of tenants.
- Isolation depends on correct RLS policies. They are reviewed like code and
  covered by tests.
- Per-tenant backup/restore or data residency would need extra work. That is
  acceptable for the MVP.
