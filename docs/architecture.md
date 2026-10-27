# CompanyCentral architecture

> **Status of this document:** target architecture. It describes what we are
> building, not what already runs. Implementation status per requirement is
> tracked in [rso-requirements.md](rso-requirements.md).

## 1. Purpose

**CompanyCentral** is a multi-tenant, cloud-native platform for small and
medium-sized manufacturing companies. A company (tenant) signs in to one Hub,
manages its users and memberships, switches on the applications it needs and
runs its shop-floor work in them: planning work orders, reserving and issuing
material, and recording production.

The MVP covers one end-to-end flow (section 9) across five application
services, an API gateway and a web frontend. It is deliberately not a complete
ERP; further applications plug into the same platform later.

## 2. System overview

```
                         ┌──────────────────────── Kubernetes cluster ────────────────────────┐
                         │                                                                     │
Browser ──HTTPS──▶ Ingress ──┬── /       ──▶ web (static React build, nginx)                   │
                         │   └── /api/*  ──▶ gateway ──REST──┬──▶ service-identity   ──▶ identity DB
                         │                                   ├──▶ service-access     ──▶ access DB
                         │                                   ├──▶ service-workflow   ──▶ workflow DB
                         │                                   ├──▶ service-inventory  ──▶ inventory DB
                         │                                   └──▶ service-production ──▶ production DB
                         │                                                                     │
                         │   services ──outbox──▶ RabbitMQ (topic exchange) ──▶ consuming services
                         └─────────────────────────────────────────────────────────────────────┘
```

- Only the Ingress is publicly reachable. Behind it, `web` serves the UI and
  `gateway` is the only backend entry point. Services are cluster-internal.
- Every service is a separately built, versioned and deployed container.
- Infrastructure (Compose, Kubernetes, Helm, monitoring) lives in
  [`deploy`](https://github.com/RSOcc26/deploy), separate from application code.

## 3. Services and data ownership

Every service owns its PostgreSQL database ([ADR-0001](adr/0001-multi-tenancy.md)).
No service reads or writes another service's database; other services use its
REST API or consume its events.

| Service | Owns | Does **not** own |
|---|---|---|
| [`service-identity`](https://github.com/RSOcc26/service-identity) | Users, companies (tenants), memberships, invitations, identity-related settings | Permissions, roles |
| [`service-access`](https://github.com/RSOcc26/service-access) | Application registry, applications enabled per tenant, roles, permissions, access groups | Users and companies (references them by ID) |
| [`service-workflow`](https://github.com/RSOcc26/service-workflow) | Catalog of articles and operations, projects, work orders, versioned operation graphs, work-order release | Stock, execution progress |
| [`service-inventory`](https://github.com/RSOcc26/service-inventory) | Stock locations, stock levels, material reservations, material issuing, stock movements | Work orders (references them by ID) |
| [`service-production`](https://github.com/RSOcc26/service-production) | Production execution, operation status, produced quantities, scrap and rejected quantities, work-order completion | Work-order definitions, stock |

Supporting repositories:

| Repository | Role |
|---|---|
| [`web`](https://github.com/RSOcc26/web) | React frontend: Hub shell, company switcher, app navigation, shared UI kit, one UI module per application |
| [`gateway`](https://github.com/RSOcc26/gateway) | Single API entry point: authentication, tenant context validation, routing, rate limiting, correlation IDs |
| [`deploy`](https://github.com/RSOcc26/deploy) | Docker Compose, Kubernetes manifests, Helm charts, environment values, monitoring |
| [`.github`](https://github.com/RSOcc26/.github) | Organization conventions, templates, reusable CI workflows, this document |

Data a service needs from another service is either requested when needed
(REST) or kept as a local, read-only copy built from events. Such copies hold
only the fields the service needs, and the owning service stays the source of
truth. Example: Production keeps a snapshot of the released operation graph
taken from the `work-order.released` event, so it can run even if Workflow is
temporarily unavailable.

## 4. Synchronous communication (REST)

- REST/JSON over HTTP inside the cluster; every service documents its API in an
  **OpenAPI 3.1** document served at `/api-docs`.
- REST is used for **questions and commands that need an immediate answer**:
  "is this user a member of this company?", "issue this material now".
- Public paths: `/api/<service>/v1/...` on the gateway, which forwards to the
  service's internal `/v1/...`. Breaking changes need a new version (`v2`).
- Every call has a timeout. Retries are only for idempotent requests, with
  backoff. Callers fail fast when a dependency is unavailable. A circuit
  breaker is planned once the services exist.
- Errors use `application/problem+json` (RFC 9457).

Every service exposes:

| Endpoint | Purpose |
|---|---|
| `GET /health/live` | Liveness: the process runs. Never checks dependencies. |
| `GET /health/ready` | Readiness: database reachable, migrations applied, broker connected. |
| `GET /metrics` | Prometheus text format. |
| `GET /api-docs` | The service's OpenAPI 3.1 document. |

## 5. Asynchronous communication (events)

- **RabbitMQ** carries domain events.
- Events state **facts that already happened** ("work order released"). They
  are not commands.
- **Transactional outbox:** a service writes the event to its own `outbox` table
  in the same database transaction as the state change. A relay publishes
  pending rows to RabbitMQ and marks them sent. Delivery is therefore
  at-least-once.
- **Idempotent consumers:** each consumer records processed event IDs
  (`inbox` table) and ignores duplicates. Messages that keep failing go to a
  dead-letter queue after a bounded number of retries.
- Topology: one durable topic exchange `companycentral.events`. The routing key
  is the event type. Each consuming service has its own durable queue per
  purpose, for example `production.work-orders`.

Event envelope (JSON):

```json
{
  "id": "0192f8e0-…",
  "type": "workflow.work-order.released",
  "version": 1,
  "occurredAt": "2026-10-23T08:15:00Z",
  "tenantId": "c0a8…",
  "correlationId": "req-…",
  "producer": "service-workflow",
  "data": { }
}
```

Initial event catalog. Payloads are defined in each service's `docs/events.md`
when implemented.

| Event | Producer | Main consumers |
|---|---|---|
| `identity.company.created` | identity | access |
| `identity.membership.added` / `.removed` | identity | access, gateway (cache invalidation) |
| `access.application.enabled` / `.disabled` | access | web (via API), interested services |
| `access.role.assigned` / `.revoked` | access | gateway (cache invalidation) |
| `workflow.work-order.released` | workflow | production, inventory |
| `workflow.work-order.cancelled` | workflow | production, inventory |
| `inventory.reservation.confirmed` / `.failed` | inventory | production |
| `inventory.material.issued` | inventory | production |
| `production.operation.completed` | production | workflow, analytics (later) |
| `production.work-order.completed` | production | workflow, inventory, analytics (later) |

## 6. Multi-tenancy

See [ADR-0001](adr/0001-multi-tenancy.md).

- **Shared, tenant-scoped tables.** Every tenant-owned row has a `tenant_id`
  column.
- **Row-level security** in PostgreSQL as a second line of defense. The service
  sets `app.tenant_id` per transaction (`SET LOCAL`), and RLS policies restrict
  every tenant-owned table to that tenant. The application role is not the table
  owner and has no `BYPASSRLS`.
- **Separate credentials** per service and per purpose: a migration role that
  owns the schema, and a runtime role with only the privileges it needs.
- **Tenant context is never trusted from the client.** The browser may *ask*
  for a company (for example with an `X-Tenant-Id` header or a route
  parameter). The gateway checks that the authenticated user is an active
  member of that company and only then forwards a signed tenant context to the
  services. Services verify that signature and take the tenant from it, never
  from raw headers.
- Events carry `tenantId`. Consumers process each event in that tenant's
  context.

## 7. Gateway and authentication boundaries

| Concern | Where |
|---|---|
| TLS termination | Ingress (cert-manager certificates) |
| Authenticating the user (credentials, sessions or tokens) | Keycloak (external OIDC provider), see [ADR-0002](adr/0002-identity-provider.md) |
| Validating the session/token on every request | gateway |
| Validating tenant membership and building the tenant context | gateway (membership from Identity, cached briefly) |
| Coarse authorization ("may this user use this application in this tenant?") | gateway, asking Access |
| Fine-grained, domain authorization ("may this user release this work order?") | the owning service, using roles from the tenant context or Access |
| Rate limiting, request size limits, correlation IDs, timeouts | gateway |

The gateway removes client-supplied identity and tenant headers before
forwarding. Whether the gateway stays custom (Node.js) or becomes Kong or
Traefik is evaluated in the gateway repository
(`docs/request-handling.md`). The gateway starts as a thin custom Node.js
service.

Only **one** authentication system is implemented: Keycloak, as decided in
ADR-0002. CompanyCentral stores no passwords or credentials.

## 8. Deployment

| Environment | How | Purpose |
|---|---|---|
| Local | Docker Compose in `deploy/compose`: PostgreSQL, RabbitMQ, Keycloak and the services | Development, integration tests |
| Cloud (dev / prod) | Kubernetes + Helm charts from `deploy/helm`, images from GHCR | Public deployment required by RSO |

- Images: `ghcr.io/rsocc26/<repository>:<version>`, built by GitHub Actions.
- Configuration through environment variables: ConfigMaps for settings,
  Secrets for credentials. Secrets are never committed.
- Probes use `/health/live` and `/health/ready`. Prometheus scrapes `/metrics`.
  Logs are structured JSON on stdout, collected centrally.
- The cloud provider is not chosen yet. The manifests avoid provider-specific
  resources so any managed Kubernetes works. No paid resources are provisioned
  without separate approval.

Target CI/CD pipeline: lint and test → build → container image → publish to
GHCR → deploy with Helm. The first two steps run through the organization's
reusable CI workflow, and images are published to GHCR by the reusable publish
workflow once a repository has a `Dockerfile`. The Helm deploy step is added
when a cluster exists.

## 9. Main end-to-end scenario

> A planner creates and releases a work order. Production receives the event
> and initializes execution. Inventory reserves and issues materials. A worker
> performs operations and records produced quantities and scrap. Downstream
> analytics can consume events to produce reports.

```
Planner (web) ──REST──▶ gateway ──▶ workflow: create work order, select operation graph version
Planner (web) ──REST──▶ gateway ──▶ workflow: release work order
                                     └─ outbox ─▶ workflow.work-order.released
                                                     ├──▶ production: create execution, operations "pending"
                                                     └──▶ inventory: reserve the required material
                                                            └─ outbox ─▶ inventory.reservation.confirmed / .failed
                                                                           └──▶ production: mark material ready / blocked
Worker (web) ──REST──▶ gateway ──▶ production: start operation
                                     └─ REST ─▶ inventory: issue reserved material (synchronous, needs an answer)
                                                  └─ outbox ─▶ inventory.material.issued
Worker (web) ──REST──▶ gateway ──▶ production: record good quantity and scrap, complete operation
                                     └─ outbox ─▶ production.operation.completed
Last operation done ───────────────▶ production.work-order.completed
                                       ├──▶ workflow: mark work order completed
                                       ├──▶ inventory: release leftover reservations
                                       └──▶ analytics (later): reports
```

Every step runs in the tenant context checked by the gateway. Every event
carries the tenant ID and the correlation ID of the request that caused it.

## 10. Observability

- Structured JSON logs to stdout with `timestamp`, `level`, `service`,
  `tenantId`, `correlationId` and `message`. Logs never contain secrets or
  personal data beyond IDs.
- `X-Request-Id` is generated by the gateway if missing, propagated on every
  REST call and copied into events as `correlationId`.
- Prometheus metrics per service: HTTP request rate, errors and latency,
  outbox backlog, consumer lag and dead-lettered messages.

## 11. Decisions and future options

Architecture decision records: [docs/adr](adr/). Service-specific decisions
go in each repository's `docs/adr/`.

Not part of the MVP; to be reconsidered when there is a concrete need:

- An activity log, a Hub mailbox and shared tasks as additional platform services
- Kafka for event replay or high-volume streams
- gRPC between services
- GraphQL aggregation for the frontend
- Event sourcing for production execution
- Runtime microfrontends or module federation
- A separate analytics service with a CQRS read model, fed by the events above
- Terraform for cluster and DNS provisioning, once a cloud provider is chosen
