# 5. Continuous deployment

*2026-11-24 · .github#26, .github#30, web#12, gateway#17, service-identity#13*

Goal: a merge to `main` of a workload repository deploys that workload, and
only that one, to `companycentral-dev` without anyone running Helm by hand.

Result: web, the gateway and `service-identity` deploy themselves. The first
runs after merging the PRs above all ended with `Deploy to dev` succeeding.

```text
 merge to main ──▶ checks ──▶ publish ──▶ deploy-dev
                              │            │  az login (GitHub OIDC, no secret)
                              ▼            │  helm upgrade <repo> --set image.tag=sha-<commit>
                 ghcr.io/rsocc26/<repo>:sha-<commit> ◀── pulled by the cluster
```

## The pieces

| Piece | Where | Chapter |
|---|---|---|
| Image `ghcr.io/rsocc26/<repo>:sha-<full commit>` | `reusable-publish.yml` | 1 |
| Identity `rso-github-actions`, federated credentials, org variables `AZURE_*` | `deploy/terraform/github-identity.tf` | 3 |
| Helm chart and values | `deploy/helm/` (read from `deploy` **`main`**) | 4 |
| The rollout | `reusable-deploy.yml` | this one |

## Step 1: The shared deploy workflow

`.github/workflows/reusable-deploy.yml` in this repository (.github#30), called
with `environment: dev` or `prod`:

1. runs in the GitHub environment of that name;
2. checks out `RSOcc26/deploy`;
3. signs in to Azure with `azure/login`;
4. asks Azure whether `rso-cluster` is running;
5. if it is, downloads a kubeconfig and runs

   ```sh
   helm upgrade --install <repo> deploy/helm/charts/service -n companycentral-<env> \
     -f deploy/helm/values/common/<repo>.yaml -f deploy/helm/values/<env>/<repo>.yaml \
     --set image.tag=sha-<commit> --wait --timeout 5m --rollback-on-failure
   ```

- **What it does:** upgrades the release named after the calling repository
  to the image built from the same commit.
- **Why:**
  - **No secret exists.** GitHub issues the job a short-lived OIDC token whose
    subject names the repository and the environment
    (`repo:RSOcc26@<org-id>/<repo>@<repo-id>:environment:dev`). Azure accepts
    it only if a federated credential with exactly that subject exists
    (chapter 3), so only our seven repositories, and only in a job with
    environment `dev` or `prod`, can sign in.
  - **The image tag is the commit.** `sha-<commit>` is immutable, so
    `helm history` shows which commit runs and a rollback restores exactly
    that image. The moving `main` tag stays for manual deploys.
  - **A stopped cluster is not a failure.** ADR-0004 runs the cluster on
    demand; the job then ends with a warning, and the next merge after
    `az aks start` deploys the latest commit.
  - **`--rollback-on-failure`:** a release that is not ready within 5 minutes
    is rolled back to the previous revision, and the job fails.
  - **One rollout at a time:** `concurrency` per repository and environment
    queues a second merge instead of running two upgrades at once.
- **If done differently:** a long-lived client secret or kubeconfig stored as
  a GitHub secret would work too, but it never expires and leaks with any
  workflow that can read it. The university tenant also blocks app
  registrations (chapter 3).

## Step 2: Call it from each workload

In `.github/workflows/ci.yml` of `web`, `gateway` and `service-identity`,
after the `publish` job:

```yaml
  deploy-dev:
    needs: publish
    if: needs.publish.outputs.image != ''
    permissions:
      contents: read
      id-token: write
    uses: RSOcc26/.github/.github/workflows/reusable-deploy.yml@main
    with:
      environment: dev
```

- **What it does:** runs the deploy only when `publish` pushed an image,
  which happens on pushes to `main` only, never on pull requests.
- **Why `id-token: write`:** without it GitHub does not issue the OIDC token
  and `azure/login` fails.
- **If skipped:** images are published but nothing rolls them out; the
  cluster keeps the previous version until someone runs Helm.
- **Order matters:** the job reads Helm values from `deploy` `main`. We merged
  deploy#15 (Keycloak and OIDC values, chapter 4) **before** these PRs;
  the other way round, the first automatic rollout would have dropped the
  OIDC settings and been rolled back.
- **The other four services** get the same job together with their
  container image.

GitHub created the `dev` environment on the first run. It has no protection
rules; `prod` will get a required reviewer when it exists.

## Step 3: Watch a rollout

```sh
gh run watch --repo RSOcc26/web
helm history web -n companycentral-dev --max 3
kubectl get pods -n companycentral-dev
```

- **What it does:** follows the run started by the merge, then shows the new
  Helm revision (chart `service-0.1.0`, status `deployed`) and the fresh pod.
  The run summary ends with "Deployed `web:sha-<commit>` to
  `companycentral-dev`".
- **If the deploy fails:**
  - `AADSTS70021` (no matching federated identity record): the token's
    subject has no federated credential. A renamed or recreated repository
    gets a new repository ID; update `deploying_repos` in
    `deploy/terraform/github-identity.tf` and apply.
  - Helm timed out and rolled back: the new pods never became ready. Read
    `kubectl logs -n companycentral-dev deploy/<repo>` and
    `kubectl describe pod -n companycentral-dev -l app.kubernetes.io/name=<repo>`.

## What is still manual

CD covers the seven workload releases only. These are upgraded by hand with
Helm (chapter 4), after their change is merged in `deploy`:

- `platform` (PostgreSQL, Kafka), `keycloak` and `edge`;
- a change to a workload's values in `deploy` alone: it takes effect with
  that workload's next merge, or right away with
  `scripts/deploy.sh dev <workload>`, which deploys the moving `main` tag.

## Still open

- `prod`: environment with a required reviewer, a release tag instead of
  `sha-<commit>`, and the deploy job on version tags.
- Deploy on changes to `deploy` itself (values and charts), so a values-only
  change does not wait for the next merge of the workload.
