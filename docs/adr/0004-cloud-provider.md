# 4. Cloud provider and cost

- **Status:** Accepted
- **Date:** 2026-11-22 (proposed and accepted)

## Context

The RSO course requires a public deployment on Kubernetes (requirements M5,
O2, O3). The project application named Azure AKS on student credits. No paid
resource may be provisioned without a recorded decision (architecture §8).

The decision is driven by money more than by features:

- **Azure for Students** gives **USD 100 once** per student (12 months, no
  card), not a monthly allowance. Two of us means at most USD 200, in two
  separate subscriptions.
- The DigitalOcean credit in the GitHub Student Developer Pack ended in
  2026, so it is no longer an option.
- Our stack is heavier than the services alone: Kafka (Strimzi), PostgreSQL
  (CloudNativePG), Keycloak, seven workloads, Traefik, cert-manager and the
  monitoring stack (Prometheus, Grafana, Loki, Alloy).

### What the cluster needs

Memory requests from `deploy/helm` values, plus estimates for components that
have no values yet:

| Part | Memory request |
|---|---|
| 5 services + gateway (128 Mi each) + web (32 Mi) | ~0.8 GiB |
| PostgreSQL 256 Mi, Kafka 1 GiB, Keycloak (estimate) ~0.75 GiB | ~2 GiB |
| Operators, Traefik, cert-manager (estimate) | ~0.8 GiB |
| Monitoring: Prometheus, Grafana, Loki, Alloy (estimate) | ~2 GiB |
| **One environment** | **~5.6 GiB** |

AKS reserves part of each node for the system, so a node with 8 GiB offers
roughly 5.5–6 GiB to pods. One environment needs **two 8 GiB nodes**;
a second full environment (its own Kafka, PostgreSQL and Keycloak) does not fit
without a third.

## Options

Prices are pay-as-you-go list prices in USD from public price trackers on
2026-11-22, compute only unless noted. **Confirm them in each provider's
pricing calculator before provisioning.**

**A. Azure AKS, Free tier.** Control plane free (no uptime SLA). Two
`Standard_B2ms` nodes (2 vCPU, 8 GiB) in West Europe: about USD 0.096 per hour
each, **~USD 140/month** running all the time. A Standard load balancer, a
public IP and disks add roughly USD 25–30/month, and they keep billing while
the cluster is stopped.

- Matches the project application; no card needed while the credit lasts.
- **Always on, USD 100 lasts about 17 days.**
- `az aks stop` / `az aks start` deallocates the nodes: 40 running hours a
  month cost about USD 8 in compute plus the ~USD 30 baseline, so the credit
  lasts about 2.5 months per student.

**B. Google GKE, zonal Standard cluster.** The GKE free tier credit covers one
zonal cluster's management fee. New customers get USD 300 for 90 days (card
required). Two `e2-standard-2` nodes (2 vCPU, 8 GiB) cost about the same as
option A.

- Three times the credit, but it expires after 90 days and needs a card, and
  the project application names Azure.

**C. AWS EKS.** USD 0.10 per hour (~USD 73/month) for the control plane alone,
on top of the nodes; AWS Educate credits are USD 35–100.

- The most expensive for us; not considered further.

## Decision

**Option A, Azure AKS Free tier, run on demand.** Azure is also the provider
recommended for the course.

- **Cluster:** one AKS cluster, Free tier, region West Europe, one node pool of
  two `Standard_B2ms` nodes, Kubernetes 1.32 or newer (Strimzi requires it).
- **Environments:** one environment (`dev`) runs in the cluster. `prod` is a
  second namespace with its own Helm values, deployed for presentations and the
  final submission only, when we can add a third node for the event.
- **Data:** PostgreSQL and Kafka stay in the cluster (CloudNativePG, Strimzi),
  as already designed. Managed Azure Database for PostgreSQL would add a
  monthly fee per server and make the manifests provider-specific.
- **Running hours:** the cluster is stopped when nobody works on it
  (`az aks stop`) and started for work sessions, demos and grading. CD
  (#26) deploys only while it is running.
- **Budget:** the student credits only (USD 100 per student); no payment
  method is added. Each subscription gets an Azure budget alert at 50 % and
  90 % of its credit. When one
  student's credit runs out, the cluster is rebuilt in the other student's
  subscription from `deploy/` (everything is in Git; data is test data).
- **DNS:** no paid domain. Hostnames under `sslip.io`, which resolves
  `<name>.<ip>.sslip.io` to the IP in the name, on Traefik's load balancer IP;
  certificates from Let's Encrypt (cert-manager, HTTP-01). A new load balancer
  IP (after a rebuild) means new hostnames, updated in the Helm values and the
  Keycloak redirect URIs.

## Consequences

- No monthly cost while the credits last, but **the cluster is not always
  reachable**. Graders and demos need it started ahead of time; that is a
  documented step (`deploy/docs/deployment.md`).
- The credits do not cover leaving the cluster on, a third node, or managed
  databases. Any of those needs a new decision and a payment method.
- Free tier has no API-server SLA; acceptable for a course project.
- Rebuilding the cluster must be routine, which favors finishing the
  Terraform option (requirement A3) and keeping all configuration in Git.
- The manifests stay provider-neutral (standard `Ingress`, default
  StorageClass), so moving to option B later costs little.
