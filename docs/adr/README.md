# Architecture decision records

Cross-cutting decisions for CompanyCentral that involve a real trade-off.
Stack defaults (Node.js, React, PostgreSQL, REST/OpenAPI, RabbitMQ, Kubernetes,
Helm, GHCR) are described in [architecture.md](../architecture.md) instead.

Each record states its context, the options, the decision and the
consequences. Status is *Proposed* or *Accepted*.

| ADR | Title | Status |
|---|---|---|
| [0001](0001-multi-tenancy.md) | Multi-tenant data isolation | Accepted |
| [0002](0002-identity-provider.md) | Identity provider: custom or external OIDC | Accepted |
