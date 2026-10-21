# CompanyHub

**A multi-tenant, cloud-native microservices platform for running a company's
work in one place** — users and companies, app access, a shared mailbox and
tasks, and business apps built on top of them.

Built for the *Cloud Computing Services* course (Računalniške storitve v oblaku,
RSO 2026/27) at the Faculty of Computer and Information Science, University of Ljubljana.

## Architecture

```
Browser ──▶ Ingress ──┬── /       ──▶ web
                      └── /api/*  ──▶ gateway ──▶ services ──▶ PostgreSQL
                                                     (one database per service)
```

- Every service owns its data and is deployed independently on Kubernetes.
- Each company (tenant) is isolated in its own database schema.
- Services talk through internal APIs and events; only the gateway is exposed.

## Repositories

| Repository | Description |
|---|---|
| [web](https://github.com/RSOcc26/web) | Frontend: the Hub shell and every app's user interface |
| [gateway](https://github.com/RSOcc26/gateway) | API gateway: authentication, tenant context, routing, rate limits |
| [service-identity](https://github.com/RSOcc26/service-identity) | Users, companies, memberships, invitations, sessions |
| [service-access](https://github.com/RSOcc26/service-access) | Which apps each company and user may use, and with which role |
| [service-audit](https://github.com/RSOcc26/service-audit) | The activity log |
| [service-messaging](https://github.com/RSOcc26/service-messaging) | The Hub mailbox |
| [service-tasks](https://github.com/RSOcc26/service-tasks) | The shared task model |

## How we work

- `main` is protected: changes land only through reviewed pull requests, squash-merged.
- Commits follow [Conventional Commits](https://www.conventionalcommits.org/); each repository keeps a `CHANGELOG.md`.
- See [CONTRIBUTING.md](https://github.com/RSOcc26/.github/blob/main/CONTRIBUTING.md) for details.

## Team

[@Gregx12](https://github.com/Gregx12) · [@jp7316](https://github.com/jp7316)
