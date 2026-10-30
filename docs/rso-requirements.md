# RSO 2026/27 requirements traceability

How CompanyCentral covers the *Cloud Computing Services* (RSO 2026/27)
requirements. The requirement list follows the team's summary of the official
assessment criteria. Check it against the official course materials before
each submission.

**Status legend:**

- **Planned:** designed and documented, nothing in the repositories yet.
- **Scaffolded:** repository structure, contracts, configuration or pipeline
  skeleton exist, but no working feature.
- **Implemented:** works and has been verified as described in the
  *Verification* column. Only mark a row Implemented with a link to the evidence
  (pull request, CI run, URL).

*Last reviewed: 2026-10-23. At this date, no row is Implemented.*

## Mandatory

| # | Requirement | Repositories | Intended implementation | Verification | Status |
|---|---|---|---|---|---|
| M1 | At least two microservices per team member (four for our team) | `service-identity`, `service-access`, `service-workflow`, `service-inventory`, `service-production` | Five independently deployable Node.js services, each with its own image and Helm release | Five Deployments running in the cluster; each image built by CI | Scaffolded |
| M2 | Graphical user interface | `web` | React Hub shell plus UI modules for Workflow, Inventory and Production | Demo of the end-to-end scenario in a browser on the public URL | Scaffolded |
| M3 | Database per service | all services, `deploy` | One PostgreSQL database and credentials per service ([ADR-0001](adr/0001-multi-tenancy.md)) | Each service's config points only to its own database; no shared credentials in `deploy` values | Planned |
| M4 | Communication between microservices | `service-workflow`, `service-inventory`, `service-production`, `gateway` | REST (gateway → services, production → inventory) and Kafka events with outbox | Releasing a work order creates an execution in Production and a reservation in Inventory; visible in logs with one correlation ID | Planned |
| M5 | Deployment on Kubernetes | `deploy` | Helm chart per service, platform manifests for PostgreSQL and Kafka | `helm list` and `kubectl get pods` show all releases healthy | Scaffolded |
| M6 | Publicly accessible cloud deployment | `deploy` | Managed Kubernetes cluster (provider not chosen) with Ingress, DNS and TLS | Public HTTPS URL reachable from outside; certificate valid | Planned |
| M7 | Technical documentation | all, especially `.github/docs` | Architecture document, ADRs, per-service READMEs, OpenAPI documents, this table | Documents reviewed before submission; links resolve | Scaffolded |

## Prioritized optional

| # | Requirement | Repositories | Intended implementation | Verification | Status |
|---|---|---|---|---|---|
| O1 | OpenAPI documentation | all services, `gateway` | `src/api/openapi.yaml` (OpenAPI 3.1) per service, served at `/api-docs`; gateway aggregates | `GET /api-docs` on every service; contract lint in CI | Scaffolded |
| O2 | CI/CD | `.github`, all repositories | Reusable workflows: lint/test/build, image build, publish to GHCR, Helm deploy | Green checks on pull requests; image in GHCR; deploy job log | Scaffolded (shared CI and image publish workflows; deploy planned) |
| O3 | Helm charts | `deploy` | Shared service chart with per-service, per-environment values | `helm lint` and `helm template` in CI; successful install | Scaffolded |
| O4 | Multi-tenancy | all services, `gateway` | Shared tables with `tenant_id` and PostgreSQL RLS; gateway validates membership ([ADR-0001](adr/0001-multi-tenancy.md)) | Integration tests: tenant A cannot read or modify tenant B's rows; forged tenant header is rejected | Planned |
| O5 | Health checks | all services, `gateway`, `deploy` | `/health/live`, `/health/ready`; Kubernetes liveness and readiness probes | Probes configured in chart; readiness fails when DB is down | Scaffolded (contracts and chart probes) |
| O6 | Fault tolerance | all services, `gateway` | Timeouts, retries for idempotent calls, outbox, idempotent consumers, dead-letter topics, circuit breaker in gateway | Demo: stop Inventory, release a work order, start Inventory, it catches up from its committed offset | Planned |
| O7 | Configuration management | all services, `deploy` | 12-factor env vars; `.env.example`; ConfigMaps and Secrets from Helm values | Same image runs locally and in the cluster with different values | Scaffolded |
| O8 | Event streaming / messaging | `service-workflow`, `service-inventory`, `service-production`, `deploy` | Kafka topics per service, outbox relay, consumer groups | Events visible on the topics; consumer lag metrics; replay after a service outage | Planned |
| O9 | Centralized logging | all services, `deploy` | Structured JSON logs to stdout, collected by Grafana Alloy into Loki | Search logs of one request across services by correlation ID | Scaffolded (deploy config only) |
| O10 | Metrics | all services, `deploy` | `/metrics` (Prometheus format), kube-prometheus-stack, ServiceMonitors | Grafana dashboard showing request rate and errors per service | Scaffolded (deploy config only) |

## Additional point opportunities

| # | Requirement | Repositories | Intended implementation | Verification | Status |
|---|---|---|---|---|---|
| A1 | API gateway | `gateway` | Custom Node.js gateway; Kong/Traefik evaluated in `gateway/docs/request-handling.md` | All `/api/*` traffic passes the gateway; rate limit returns `429` | Scaffolded |
| A2 | OAuth2/OIDC-based IAM | `service-identity`, `gateway`, `deploy` | Keycloak as OIDC provider ([ADR-0002](adr/0002-identity-provider.md)); authorization code flow with PKCE in `web`; gateway validates tokens against Keycloak's JWKS | Login via OIDC flow; gateway validates tokens | Planned |
| A3 | Terraform | `deploy` | Optional `deploy/terraform` for cluster and DNS once a provider is chosen | `terraform plan` in CI | Planned |
| A4 | Intermediate project presentation | n/a | Present the architecture, the deployed skeleton and the first flow | Presentation held | Planned |
