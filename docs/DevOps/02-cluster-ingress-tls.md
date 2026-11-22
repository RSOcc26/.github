# 2. Cluster access, Ingress and TLS

*2026-11-22 · deploy#10 · decision: [ADR-0004](../adr/0004-cloud-provider.md)*

Goal: a public HTTPS address that routes `/` to web and `/api/*` to the
gateway.

Result: `https://dev.<ip>.sslip.io` (on 2026-11-22: `dev.68-210-93-25.sslip.io`)
with a trusted Let's Encrypt certificate. web and the gateway are deployed in
a later step (deploy#12), so until then the address answers `404`.

## Before you start: tools on your machine

| Tool | Talks to | Install |
|---|---|---|
| `az` (Azure CLI) | Azure's management API | distribution package or Microsoft's installer |
| `kubectl` | the cluster's Kubernetes API | distribution package |
| `helm` | the Kubernetes API (renders charts into Kubernetes resources) | see step 3 |

None of these needs SSH, and nothing is installed *on* the cluster to make
them work. Helm 2 had an in-cluster component (Tiller); Helm 3 and 4 do not.

## The cluster we found

The team already had an AKS cluster, created by hand in the Azure portal:

| Setting | Value | Matches ADR-0004? |
|---|---|---|
| Name / resource group | `rso-cluster` / `rso-rg` | |
| Region | Austria East | Yes. The Azure for Students subscription only allows Poland Central, France Central, Norway East, Austria East and Germany West Central (an Azure Policy); West Europe is blocked |
| Kubernetes | 1.36.4 | Yes (Strimzi needs 1.32+) |
| Tier | Free | Yes (Standard costs about USD 73/month) |
| Nodes | 2× `Standard_DS2_v2` (2 vCPU, 7 GiB) | Close. The ADR planned 2× B2ms (8 GiB). DS2_v2 costs about 35% more per hour, but only while running: about USD 4/month at 40 h, so we kept it |

We inspected it with:

```sh
az aks list --query "[].{name:name, rg:resourceGroup, region:location, k8s:kubernetesVersion, tier:sku.tier, power:powerState.code, pools:agentPoolProfiles[].{name:name, vm:vmSize, count:count, mode:mode}}" -o json
```

- **What it does:** lists AKS clusters in the subscription, with only the
  fields we care about (`--query` is a JMESPath filter).
- **Why:** to check the existing cluster against the ADR before building on
  it: an unexpected region, an old Kubernetes version or the paid tier would
  each have forced a change first.

## Step 1: Sign in to Azure

```sh
az login --tenant <tenant-id>
```

- **What it does:** opens a browser sign-in and stores a token for the Azure
  CLI.
- **Why:** the university's sign-in policy expires the CLI session after 30
  days (`AADSTS70043: The refresh token has expired`). Naming the tenant signs
  in to the university directory that owns the student subscription.
- **If skipped:** every `az` command fails with the error above.

## Step 2: Start the cluster and connect `kubectl`

```sh
az aks start -g rso-rg -n rso-cluster
```

- **What it does:** starts the stopped cluster: Azure allocates the two node
  VMs again and they join the cluster (about 5 minutes; it can outlast a
  2-minute terminal timeout and keep running in the background).
- **Why:** ADR-0004 runs the cluster on demand. The nodes cost about
  USD 0.27/hour together **only while running**.
- **If skipped:** the API answers, but there are no nodes, so nothing runs.

```sh
az aks get-credentials -g rso-rg -n rso-cluster
kubectl get nodes -o wide
kubectl get ns
```

- **What it does:** `get-credentials` writes the cluster's address and a
  credential into `~/.kube/config` and makes it the current context. The two
  `kubectl` commands then confirm the connection: both nodes `Ready`, and only
  the system namespaces (`default`, `kube-*`), so nobody had installed
  anything before.
- **Why:** `kubectl` and `helm` read `~/.kube/config` to know which cluster to
  talk to.
- **If skipped:** `kubectl` and `helm` have no cluster (or, worse, a different
  one from an earlier context). Check with `kubectl config current-context`.

## Step 3: Install Helm locally

```sh
V=$(curl -s https://api.github.com/repos/helm/helm/releases/latest | jq -r .tag_name)
curl -sL https://get.helm.sh/helm-$V-linux-amd64.tar.gz | tar xz -C /tmp linux-amd64/helm
mkdir -p ~/.local/bin && install -m755 /tmp/linux-amd64/helm ~/.local/bin/helm
helm version
```

- **What it does:** downloads the latest Helm release (v4.3.2 on that day)
  and installs it for the current user only, without `sudo`.
- **Why:** our charts (`deploy/helm/`) and the third-party components
  (Traefik, cert-manager) are installed with Helm.
- **If done differently:** Azure Cloud Shell in the portal has Helm
  preinstalled, but then the deploy repository has to be cloned there.

## Step 4: Namespaces and Traefik (the Ingress controller)

```sh
cd deploy
kubectl apply -f k8s/namespaces.yaml
helm repo add traefik https://traefik.github.io/charts
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade --install traefik traefik/traefik --version 42.1.3 \
  -n traefik --create-namespace --wait
```

- **What it does:**
  - creates the namespaces `companycentral-dev`, `companycentral-prod` and
    `monitoring`;
  - registers the chart repositories for Traefik and cert-manager (Jetstack);
  - installs Traefik, chart version 42.1.3, into its own namespace.
    `upgrade --install` installs the first time and upgrades later, so the
    same command is safe to repeat. `--wait` returns only when Traefik is
    running.
- **Why:**
  - An *Ingress controller* is the one public entry point. It reads
    `Ingress` resources and routes requests by host and path to the right
    Service. Traefik's Service is of type `LoadBalancer`, so Azure gives it a
    **public IP** (billable, part of the ~USD 30/month that continues while the
    cluster is stopped, see ADR-0004).
  - Versions are pinned (`--version`) so a rebuild gets exactly the same
    software. Without it Helm takes whatever is newest that day.
  - Traefik instead of *ingress-nginx*: ingress-nginx was retired by the
    Kubernetes project in 2026.
- **If skipped:** `Ingress` resources do nothing; there is no public address.

```sh
kubectl get svc -n traefik traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

- **What it does:** prints the public IP Azure assigned (`68.210.93.25`).
  Empty for the first ~30 seconds while Azure assigns it.
- **Why:** the host name is built from it (step 5). A plain
  `curl http://<ip>/` answered `404` from Traefik: reachable, no routes yet.

## Step 5: cert-manager and the edge chart (Let's Encrypt staging)

```sh
helm upgrade --install cert-manager jetstack/cert-manager --version v1.21.2 \
  -n cert-manager --create-namespace --set crds.enabled=true --wait
```

- **What it does:** installs cert-manager, which requests and renews TLS
  certificates. `crds.enabled=true` also installs its resource types
  (`Certificate`, `ClusterIssuer`, ...).
- **If done without `crds.enabled=true`:** the edge chart in the next command
  fails, because Kubernetes does not know what a `ClusterIssuer` is.

```sh
helm upgrade --install edge helm/charts/edge -n companycentral-dev \
  -f helm/values/dev/edge.yaml \
  --set host=dev.68-210-93-25.sslip.io \
  --set clusterIssuer.email=<your-email> --wait
```

- **What it does:** installs our `edge` chart: the `Ingress` (`/` → web,
  `/api` → gateway, TLS for the host) and a `ClusterIssuer` for Let's Encrypt
  **staging** (the default in `values/dev/edge.yaml`). cert-manager sees the
  Ingress, proves to Let's Encrypt that the host reaches our cluster (the
  *HTTP-01 challenge*: Let's Encrypt fetches a token over plain HTTP) and
  stores the certificate in the Secret `companycentral-tls`.
- **Why sslip.io:** `dev.68-210-93-25.sslip.io` resolves to `68.210.93.25`
  with no DNS setup and no cost (ADR-0004: no paid domain).
- **Why staging first:** staging has generous rate limits, so mistakes cost
  nothing; its certificates are not trusted by browsers.
- **The email** is only used by Let's Encrypt for expiry notices.
- **If the IP changes** (a new LoadBalancer after a rebuild), the host name
  changes too: update `host`, and later the Keycloak redirect URIs.

Check:

```sh
kubectl get certificate,challenge -n companycentral-dev
```

`READY True` after about 40 seconds. A `challenge` row only appears while the
HTTP-01 check runs.

## Step 6: Switch dev to Let's Encrypt production

```sh
helm upgrade edge helm/charts/edge -n companycentral-dev \
  -f helm/values/dev/edge.yaml \
  --set host=dev.68-210-93-25.sslip.io \
  --set clusterIssuer.email=<your-email> \
  --set clusterIssuer.name=letsencrypt-prod \
  --set clusterIssuer.server=https://acme-v02.api.letsencrypt.org/directory --wait
```

- **What it does:** replaces the staging issuer with `letsencrypt-prod`.
  cert-manager notices the Ingress now points at a different issuer and
  requests a new, browser-trusted certificate (issued in about 40 seconds).
- **Why:** deploy#10 requires a *valid* certificate, and per ADR-0004 dev is
  the environment used for demos and grading. Prod runs only for events.
- **Risk we checked first:** sslip.io is **not** on the Public Suffix List, so
  Let's Encrypt may count all sslip.io users against one shared weekly limit.
  It worked; if it ever fails with `rateLimited`, the free fallback is
  `duckdns.org`, which is on the list and keeps a fixed name.
- **Not yet in the repository:** these `--set` overrides become the default in
  `helm/values/{dev,prod}/edge.yaml` (dev creates `letsencrypt-prod`, prod
  reuses it). Each `ClusterIssuer` may be created by only one release.

Check that the served certificate is trusted:

```sh
curl -s -o /dev/null -w '%{http_code} %{ssl_verify_result}\n' https://dev.68-210-93-25.sslip.io/
```

`404 0`: `0` means the certificate verified; `404` is expected until web and
the gateway are deployed.

## When you are done: stop the cluster

```sh
az aks stop -g rso-rg -n rso-cluster
```

- **What it does:** deallocates the node VMs. Everything installed stays (it
  lives in the cluster's state and disks) and comes back on `az aks start`.
- **Why:** running nodes are the largest cost. The load balancer, public IP
  and disks still cost a little while stopped.
- **If skipped:** about USD 6.50 of student credit per day, for nothing.

## Still open for deploy#10

- ~~Record how the cluster is (re)created~~: done with Terraform, see
  [chapter 3](03-terraform-and-github-identity.md).
- Access for the second team member, documented in `deploy/docs/deployment.md`.
- ADR-0004 correction: Austria East and DS2_v2 nodes.
