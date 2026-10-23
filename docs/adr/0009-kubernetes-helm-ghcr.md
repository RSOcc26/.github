# 9. Kubernetes, Helm, GHCR and a central deploy repository

- **Status:** Accepted
- **Date:** 2026-10-23

## Context

RSO requires deployment to a publicly accessible, cloud-hosted Kubernetes
cluster. Infrastructure configuration changes at a different pace and for
different reasons than service code.

## Decision

- Local development uses **Docker Compose**. Cloud deployment uses
  **Kubernetes** with **Helm** charts.
- All infrastructure configuration (Compose files, Helm charts, environment
  values, platform components, monitoring) lives in the **`deploy`**
  repository, not in service repositories.
- Images are built by **GitHub Actions** and published to the **GitHub
  Container Registry** as `ghcr.io/rsocc26/<repository>`, tagged with the
  commit SHA and with release versions.
- The pipeline grows in this order: lint/test → build → image → publish to
  GHCR → Helm deploy. The organization's reusable workflows in
  `RSOcc26/.github` implement each stage once for every repository.
- Configuration is passed as environment variables. Secrets come from
  Kubernetes Secrets created outside Git (later: sealed or external secrets).
- Manifests avoid cloud-provider-specific resources. The provider is chosen
  separately, and nothing paid is provisioned without explicit approval.

## Consequences

- A service release is: merge → image published → bump the image tag in
  `deploy` → Helm upgrade.
- Terraform for the cluster itself is optional and added only when a provider
  is chosen.
