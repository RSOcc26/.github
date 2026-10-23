# Architecture decision records

Cross-cutting decisions for CompanyCentral. Decisions that affect only one
repository live in that repository's `docs/adr/`.

Format: [Michael Nygard's template](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)
(Context, Decision, Consequences). Status is one of *Proposed*, *Accepted*,
*Superseded by …* or *Deprecated*. Accepted records are not edited, only
superseded by a new record.

| ADR | Title | Status |
|---|---|---|
| [0001](0001-record-architecture-decisions.md) | Record architecture decisions | Accepted |
| [0002](0002-service-decomposition.md) | Service decomposition and first milestone | Accepted |
| [0003](0003-database-per-service.md) | PostgreSQL, one database per service | Accepted |
| [0004](0004-multi-tenancy-shared-tables-rls.md) | Multi-tenancy with shared tables, `tenant_id` and row-level security | Accepted |
| [0005](0005-rest-and-openapi.md) | REST/JSON with OpenAPI 3.1 for synchronous calls | Accepted |
| [0006](0006-rabbitmq-transactional-outbox.md) | RabbitMQ with a transactional outbox for events | Accepted |
| [0007](0007-identity-provider-open-decision.md) | Custom identity management or an external OIDC provider | **Proposed (open)** |
| [0008](0008-tenant-context-validation.md) | Tenant context is validated by the gateway, never trusted from clients | Accepted |
| [0009](0009-kubernetes-helm-ghcr.md) | Kubernetes, Helm, GHCR and a central deploy repository | Accepted |
| [0010](0010-node-react-stack.md) | Node.js LTS backends and one React frontend | Accepted |

New record: copy the next number, keep the title short, and add it to this table.
