# CompanyCentral

**A multi-tenant, cloud-native platform for small and medium-sized
manufacturing companies.** One Hub for users and companies, with applications
switched on per company: work-order planning, inventory and production
execution.

Built for the *Cloud Computing Services* course (Računalniške storitve v oblaku,
RSO 2026/27) at the Faculty of Computer and Information Science, University of Ljubljana.

> **Status:** architecture defined, repositories scaffolded. No service is
> implemented yet. See the [requirements traceability](https://github.com/RSOcc26/.github/blob/main/docs/rso-requirements.md).

## Architecture

```
Browser ──▶ Ingress ──┬── /       ──▶ web
                      └── /api/*  ──▶ gateway ──REST──▶ services ──▶ PostgreSQL
                                                          │        (one database per service)
                                                          └──events──▶ RabbitMQ
```

- Every service owns its data and is deployed independently on Kubernetes with Helm.
- Tenants share tables, isolated by `tenant_id` and PostgreSQL row-level security.
- REST (OpenAPI 3.1) for questions, events with a transactional outbox for facts.
- Only the gateway is exposed; it validates the user's membership in the company before anything reaches a service.

Details: [architecture](https://github.com/RSOcc26/.github/blob/main/docs/architecture.md) ·
[decisions (ADRs)](https://github.com/RSOcc26/.github/tree/main/docs/adr)

## Repositories

| Repository | Description |
|---|---|
| [web](https://github.com/RSOcc26/web) | React frontend: Hub shell, company switcher, application UIs |
| [gateway](https://github.com/RSOcc26/gateway) | API gateway: authentication, tenant context, routing, rate limits |
| [service-identity](https://github.com/RSOcc26/service-identity) | Users, companies, memberships, invitations |
| [service-access](https://github.com/RSOcc26/service-access) | Application registry, applications per company, roles, permissions, access groups |
| [service-workflow](https://github.com/RSOcc26/service-workflow) | Catalog of articles and operations, projects, work orders, release |
| [service-inventory](https://github.com/RSOcc26/service-inventory) | Stock locations, material reservations, issuing, stock movements |
| [service-production](https://github.com/RSOcc26/service-production) | Production execution, quantities, scrap, work-order completion |
| [deploy](https://github.com/RSOcc26/deploy) | Docker Compose, Kubernetes, Helm charts, monitoring |
| [.github](https://github.com/RSOcc26/.github) | Organization conventions, templates, reusable CI, architecture docs |

## How we work

- `main` is protected: changes land only through reviewed pull requests, squash-merged.
- Commits follow [Conventional Commits](https://www.conventionalcommits.org/); each repository keeps a `CHANGELOG.md`.
- Everyone follows the [code of conduct](https://github.com/RSOcc26/.github/blob/main/CODE_OF_CONDUCT.md).
- See [CONTRIBUTING.md](https://github.com/RSOcc26/.github/blob/main/CONTRIBUTING.md) for details.

## Team

[@Gregx12](https://github.com/Gregx12) · [@jp7316](https://github.com/jp7316)

All repositories are licensed under the [MIT License](https://github.com/RSOcc26/.github/blob/main/LICENSE).
