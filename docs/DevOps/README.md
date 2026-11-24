# DevOps log

A running record of how CompanyCentral is built, published and deployed: what
we did, the exact commands, what each command does, why we ran it, and what
would happen if we ran it differently or skipped it.

The decisions behind the setup are in the ADRs
([ADR-0004](../adr/0004-cloud-provider.md) for the cloud). The reference for
day-to-day use is [`deploy/docs/deployment.md`](https://github.com/RSOcc26/deploy/blob/main/docs/deployment.md).
This log explains *how we got here*, in order.

| # | Chapter | Issues |
|---|---|---|
| 1 | [CI pipeline and container images](01-ci-and-container-images.md) | .github#25 |
| 2 | [Cluster access, Ingress and TLS](02-cluster-ingress-tls.md) | deploy#10 |
| 3 | [Terraform and the GitHub Actions identity](03-terraform-and-github-identity.md) | deploy#10, .github#26 |
| 4 | [Platform, Keycloak and the first deploy](04-platform-keycloak-first-deploy.md) | deploy#12, deploy#15 |
| 5 | [Continuous deployment](05-continuous-deployment.md) | .github#26, .github#30 |

## How the pieces fit

```text
 developer ──push/PR──▶ GitHub Actions ──merge to main──▶ GHCR (container images)
                                                            │
 laptop: az, kubectl, helm ──HTTPS──▶ Azure (rso-cluster) ◀─┘ pulls images
                                          │
 browser ──HTTPS──▶ public IP ──▶ Traefik (Ingress) ──▶ web  (/)
                                                    ├─▶ gateway (/api/*) ──▶ services
                                                    └─▶ keycloak (/idp/*)
```

There is no SSH anywhere. `az` talks to Azure's management API, and `kubectl`
and `helm` talk to the cluster's Kubernetes API, all over HTTPS from the
machine that runs them (a laptop, or a GitHub Actions runner for CD).

## Conventions in this log

- Commands are shown as they were run. Values that differ per person or per
  rebuild are written as placeholders: `<your-email>`, `<ip>`.
- Never paste secrets, tokens or `.env` contents here; the repositories are
  public.
- Each command has four parts: **What it does**, **Why**, **If done
  differently**, **If skipped**.
