# 2. Service decomposition and first milestone

- **Status:** Accepted
- **Date:** 2026-10-23

## Context

The earlier plan (CompanyHub) started with platform services: identity, access,
audit, messaging and tasks. Business applications would come later. RSO requires
at least two microservices per team member (four for our team), a working GUI,
communication between services and database-per-service. A platform without a
business flow does not show service communication convincingly.

## Decision

The first milestone consists of:

| Repository | Responsibility |
|---|---|
| `service-identity` | Users, companies (tenants), memberships, invitations |
| `service-access` | Application registry, applications per tenant, roles, permissions, access groups |
| `service-workflow` | Catalog of articles and operations, projects, work orders, versioned operation graphs, release |
| `service-inventory` | Stock locations, reservations, issuing, stock movements |
| `service-production` | Execution of operations, quantities, scrap, completion of work orders |
| `gateway` | Single API entry point |
| `web` | React frontend |

The platform services `service-audit`, `service-messaging` and `service-tasks`
are **dropped from the plan**; their scaffold repositories (no code) were
deleted. They can be recreated later from the same conventions.

Identity and Access stay separate services: who someone is (and to which
company they belong) changes for different reasons than what they may do.

## Consequences

- Five application services give a margin above the required four.
- The work-order flow (release → reserve → execute → complete) exercises both
  REST and events across three services.
- Audit, messaging or tasks can be added later as new services. No
  first-milestone service depends on them.
