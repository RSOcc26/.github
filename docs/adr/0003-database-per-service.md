# 3. PostgreSQL, one database per service

- **Status:** Accepted
- **Date:** 2026-10-23

## Context

Services must be deployable and changeable independently, and RSO requires
database-per-service. A shared database would couple schemas and releases.

## Decision

- Every service has its **own PostgreSQL database and its own credentials**.
  No service connects to another service's database, not even read-only.
- Data from another service is obtained through its REST API or by keeping a
  local read-only copy built from its events.
- Each service has two roles: a **migration role** that owns the schema and a
  **runtime role** with only DML privileges on its own tables.
- Schema changes are versioned migrations in the service's `migrations/`
  folder, applied before the new version starts.
- Locally and in the course cluster, the databases may share one PostgreSQL
  server (separate databases and roles). In production they could be separate
  instances without code changes.

## Consequences

- Cross-service queries (reports) are built from events, not joins.
- Referential integrity across services is eventual. Services store foreign IDs
  without foreign-key constraints.
