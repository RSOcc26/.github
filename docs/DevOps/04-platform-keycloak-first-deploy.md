# 4. Platform, Keycloak and the first deploy

*2026-11-24 · deploy#12, deploy#15 · decisions: [ADR-0002](../adr/0002-identity-provider.md), [ADR-0004](../adr/0004-cloud-provider.md)*

Goal: web, the gateway and `service-identity` running in `companycentral-dev`,
with PostgreSQL, Kafka and Keycloak in the cluster, and a working sign-in at
the public address.

Result: `https://dev.<ip>.sslip.io` serves web, `/api/*` the gateway and
`/idp/*` Keycloak. Registration and sign-in work end to end. The other four
services have no published image yet and are not deployed.

```text
 browser ──▶ Traefik ──▶ web       (/)
                    ├──▶ gateway   (/api/*) ──▶ service-identity
                    └──▶ keycloak  (/idp/*)
                 cluster-internal: PostgreSQL (CloudNativePG), Kafka (Strimzi)
```

## Before you start

The cluster is running and `kubectl` points at it (chapter 2, steps 1–2), and
the `deploy` repository is on an up-to-date `main`:

```sh
az aks start -g rso-rg -n rso-cluster
cd deploy && git switch main && git pull
```

## Step 1: The database and Kafka operators

```sh
helm repo add cnpg https://cloudnative-pg.github.io/charts && helm repo update cnpg
helm upgrade --install cnpg cnpg/cloudnative-pg --version 0.29.1 \
  -n cnpg-system --create-namespace --wait
helm upgrade --install strimzi oci://quay.io/strimzi-helm/strimzi-kafka-operator --version 1.3.0 \
  -n strimzi --create-namespace \
  --set 'watchNamespaces={companycentral-dev,companycentral-prod}' --wait
kubectl get pods -n cnpg-system
kubectl get pods -n strimzi
```

- **What it does:** installs two *operators*: programs in the cluster that
  watch for our own resource types (`Cluster`, `Database`, `Kafka`,
  `KafkaTopic`, `KafkaUser`) and create and run the real PostgreSQL and Kafka
  for them. Versions are the pinned ones from `deploy/k8s/README.md`.
- **Why:** PostgreSQL and Kafka are then declared in our `platform` chart like
  any other resource, including one database and two roles per service, and
  one Kafka user per client with access to its own topics only. No managed
  cloud database or broker, so no extra cost and no provider lock-in.
- **If done differently:** Strimzi's `watchNamespaces` must list every
  namespace that holds a `Kafka` resource. Without it the operator ignores
  them and the `platform` release waits until it times out.
- **If skipped:** the `platform` chart fails with `no matches for kind
  "Cluster"`; the resource types do not exist.

## Step 2: Secrets

```sh
scripts/create-secrets.sh companycentral-dev
kubectl get secrets -n companycentral-dev
```

- **What it does:** generates random passwords and keys and stores them only
  as Kubernetes Secrets in the namespace:
  - per database owner (`identity`, `access`, `workflow`, `inventory`,
    `production`, `keycloak`): `pg-<svc>-migrator` and `pg-<svc>-app` (role
    credentials CloudNativePG applies) and `<svc>-database` (the connection
    URLs the service reads);
  - `gateway-tenant-context-key`: the key the gateway signs the tenant
    context with;
  - `keycloak-admin`: Keycloak's bootstrap admin;
  - `identity-idp-client`: the secret of the `service-identity` Keycloak
    client, read both by Keycloak's realm import and by the service.
- **Why:** credentials never go into Git or Helm values. The script skips
  Secrets that already exist, so it is safe to run again after a new service
  is added. Kafka passwords are not created here: Strimzi generates a Secret
  `kafka-<client>` for every `KafkaUser` in step 3.
- **If skipped:** pods stay in `CreateContainerConfigError` (a referenced
  Secret does not exist).

Read a value only when you need it, for example the Keycloak admin password:

```sh
kubectl get secret keycloak-admin -n companycentral-dev -o jsonpath='{.data.password}' | base64 -d; echo
```

## Step 3: The platform release (PostgreSQL and Kafka)

```sh
helm upgrade --install platform helm/charts/platform -n companycentral-dev \
  -f helm/values/dev/platform.yaml --wait --timeout 10m
kubectl get cluster,database -n companycentral-dev
kubectl get kafka,kafkatopic,kafkauser -n companycentral-dev
```

- **What it does:** declares one PostgreSQL cluster `companycentral-pg` with
  six databases and their roles, and one Kafka cluster `companycentral-kafka`
  (KRaft, one node in dev) with the `<service>.events` and `<service>.dlt`
  topics and one SCRAM user per client. The operators from step 1 create the
  pods and disks.
- **Why:** each service owns its database (organization ADR-0001); the
  runtime role has no `BYPASSRLS`, so row-level security stays enforced.
- **If skipped:** the services cannot connect and fail their readiness check.
- **Cost:** two persistent disks (2 GiB each in dev). Disks are billed while
  the cluster is stopped too, a few cents a month at this size.

## Step 4: Keycloak

```sh
helm upgrade --install keycloak helm/charts/keycloak -n companycentral-dev \
  --set host=dev.<ip>.sslip.io --wait --timeout 10m
kubectl logs -n companycentral-dev deploy/keycloak | grep -E -i 'imported|started in'
```

- **What it does:** runs Keycloak 26.8.1 in production mode on the `keycloak`
  database, served under `https://<host>/idp`. On its first start it imports
  the `companycentral` realm and the login theme from
  `helm/charts/keycloak/files/` (the same files Compose mounts locally) and
  fills in the `service-identity` client secret and the public web address
  (`https://<host>/*` as a redirect URI) from Secrets and values.
- **Why:**
  - The browser has to reach Keycloak directly to sign in, so it needs a
    public URL. A path on the existing host reuses the Ingress and the
    certificate; a subdomain would need a second certificate.
  - One replica with the `Recreate` strategy: more replicas need Keycloak's
    distributed cache configured, and two instances must never share the
    database during an upgrade.
  - The first start takes 1–2 minutes because Keycloak builds itself for
    PostgreSQL; the startup probe allows for that.
- **If done differently:** we first used `/auth`. Registration then ended on
  Keycloak's "Page not found": web's sign-in callback is `/auth/callback`
  (`web/src/shell/auth/userManager.js`), so the Ingress sent web's own route
  to Keycloak. Check a new path prefix against the routes of web before
  claiming it.
- **If skipped:** web cannot sign in and the gateway cannot verify tokens.
- **Note:** the realm is imported only when it does not exist. Later edits
  to `companycentral-realm.json` must be made in the admin console
  (`https://<host>/idp/admin`, user `admin`) as well, or the realm deleted
  and imported again, which deletes its users.

## Step 5: Route `/idp` to Keycloak

```sh
helm upgrade --install edge helm/charts/edge -n companycentral-dev \
  -f helm/values/dev/edge.yaml \
  --set host=dev.<ip>.sslip.io --set clusterIssuer.email=<your-email>
curl -s https://dev.<ip>.sslip.io/idp/realms/companycentral/.well-known/openid-configuration | jq .issuer
```

- **What it does:** adds the rule `/idp` → `keycloak` to the Ingress next to
  `/api` → gateway and `/` → web. The `curl` prints the realm's issuer,
  `https://dev.<ip>.sslip.io/idp/realms/companycentral`.
- **Why:** the issuer is the value the gateway checks every token against
  (step 6). It is built from the public URL, so it must be the one the
  browser sees.
- **If done differently:** `helm/values/dev/edge.yaml` now selects Let's
  Encrypt production itself (chapter 2 set it with `--set`). Upgrading
  without the values file would switch dev back to the untrusted staging
  certificate. Only the dev release creates the cluster-wide
  `letsencrypt-prod` issuer; prod reuses it.

## Step 6: web, the gateway and `service-identity`

```sh
scripts/deploy.sh dev service-identity gateway web
kubectl get pods -n companycentral-dev
curl -s -o /dev/null -w '%{http_code}\n' https://dev.<ip>.sslip.io/
```

- **What it does:** `helm upgrade --install` of the generic `service` chart
  once per workload, with `helm/values/common/<name>.yaml` and
  `helm/values/dev/<name>.yaml`, image tag `main`. Naming the workloads skips
  the platform, Keycloak and edge releases.
- **Why the values matter:**
  - gateway: `OIDC_ISSUER` (public URL, as in step 5), `OIDC_AUDIENCE=gateway`
    and `OIDC_JWKS_URL` (Keycloak's keys over the cluster network,
    `http://keycloak/idp/...`).
  - web: `OIDC_AUTHORITY` (the same public realm URL) and
    `OIDC_CLIENT_ID=web`.
  - `service-identity`: the Keycloak admin API and token URLs (cluster
    network), `IDP_CLIENT_SECRET`, and `DATABASE_MIGRATION_URL`: the service
    runs its migrations itself at startup, as the migration role.
  - dev ServiceMonitors are off until kube-prometheus-stack is installed;
    with them on, Helm fails with `no matches for kind "ServiceMonitor"`.
- **If skipped:** the address keeps answering `404`.

Then, in a private browser window: open `https://dev.<ip>.sslip.io`, sign in,
register. The login pages use our theme, and after registering the browser
returns to web signed in.

## When you are done

Stop the cluster (chapter 2). Everything installed here comes back on
`az aks start`.

## Still open

- The other four services: each gets a release once its image is published.
- Monitoring: kube-prometheus-stack, then turn the dev ServiceMonitors back on.
- prod: host name, `OIDC_ISSUER` and `OIDC_AUTHORITY` in prod values, Secrets.
- The dev host is written in three places (edge and keycloak `--set host`,
  gateway and web dev values). If Traefik's public IP changes, all three
  change.
- Restrict `/idp/admin` (for example by source IP) before prod.
