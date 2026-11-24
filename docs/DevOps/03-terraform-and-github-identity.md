# 3. Terraform and the GitHub Actions identity

*2026-11-22 · deploy#10, .github#26 · code: [`deploy/terraform/`](https://github.com/RSOcc26/deploy/tree/main/terraform)*

Goal: the Azure setup lives in Git as code (course requirement A3), and
GitHub Actions can reach Azure without any stored password.

Result:

- Terraform manages `rso-rg` and `rso-cluster`. Both were adopted from the
  portal without being recreated.
- A managed identity `rso-github-actions` that GitHub workflows sign in as,
  with a short-lived token.
- Three GitHub organization variables that tell the workflows which identity
  to use.

## Step 1: Install Terraform

```sh
V=$(curl -s https://api.releases.hashicorp.com/v1/releases/terraform/latest | jq -r .version)
curl -sLo /tmp/tf.zip https://releases.hashicorp.com/terraform/${V}/terraform_${V}_linux_amd64.zip
unzip -o -q /tmp/tf.zip terraform -d ~/.local/bin && terraform version
```

- **What it does:** installs the latest Terraform (v1.17.2 on that day) for
  the current user only, like Helm in chapter 2.
- **Why Terraform:** it describes infrastructure as files and shows a *plan*
  (the exact difference between the files and reality) before changing
  anything.
- **If done differently:** a shell script of `az` commands can create a
  cluster, but it cannot tell you what has drifted, and running it twice
  fails or duplicates resources.

## Step 2: A place for Terraform's state

```sh
az provider register -n Microsoft.Storage --wait
az storage account create -g rso-rg -n rsocc26tfstate -l austriaeast \
  --sku Standard_LRS --min-tls-version TLS1_2 --allow-blob-public-access false
az storage container create --account-name rsocc26tfstate -n tfstate --auth-mode login
```

- **What it does:**
  - registers the Storage service for the subscription. A new student
    subscription has it switched off; until then every storage command fails
    with `SubscriptionNotFound`, a misleading message;
  - creates a storage account (about USD 0.05/month for one small file),
    TLS 1.2 or newer only, no public blobs;
  - creates a container `tfstate` in it, signing in with Entra ID instead of
    an access key.
- **Why:** Terraform keeps a *state* file that maps each resource in our code
  to its real Azure ID. In shared storage, both team members (and CI) see the
  same state, and Terraform *locks* it during each run.
- **Why by hand:** Terraform cannot store its state in a storage account it
  has not created yet (chicken and egg). This is the only resource made
  outside Terraform.
- **If kept locally (the default `terraform.tfstate`):** only the laptop that
  ran it knows what exists. Another machine would try to create everything
  again, and a lost laptop means re-importing everything. The file can also
  contain secrets, so it must never go to Git (`.gitignore` excludes it).

```sh
az role assignment create --assignee $(az ad signed-in-user show --query id -o tsv) \
  --role "Storage Blob Data Contributor" \
  --scope $(az storage account show -g rso-rg -n rsocc26tfstate --query id -o tsv)
```

- **What it does:** lets you read and write blobs in the state account.
- **Why:** Owner of a subscription manages storage accounts but may not read
  their *data* when signing in with Entra ID. Without this role, `terraform
  init` fails with `403 AuthorizationPermissionMismatch`. Role assignments
  take a few minutes to take effect.
- **If done differently:** access keys would work without the role, but a key
  is a shared password with full rights that never expires.

## Step 3: Adopt the existing cluster (import)

The repository's `terraform/` folder holds `versions.tf` (provider, state
location), `variables.tf` (the subscription ID) and `main.tf` with *import
blocks*:

```hcl
import {
  to = azurerm_kubernetes_cluster.main
  id = "/subscriptions/${var.subscription_id}/resourceGroups/rso-rg/providers/Microsoft.ContainerService/managedClusters/rso-cluster"
}
```

```sh
cd deploy/terraform
export TF_VAR_subscription_id=$(az account show --query id -o tsv)
terraform init
terraform plan -generate-config-out=aks.tf
```

- **What it does:**
  - `TF_VAR_subscription_id` passes the subscription ID to Terraform without
    writing it in a file (`TF_VAR_<name>` fills `variable "<name>"`);
  - `init` downloads the Azure provider (`azurerm`), connects to the state
    storage and writes `.terraform.lock.hcl`, which pins the provider's exact
    version and checksums. The lock file is committed, the `.terraform/`
    folder is not;
  - `plan -generate-config-out` reads the live cluster and writes every
    setting it finds into `aks.tf`.
- **Why import:** the cluster already ran Traefik and cert-manager (chapter
  2). Recreating it from code would delete them and change the public IP.
  Import only records the existing resources in the state.
- **The generated file needed cleaning:** it lists more than 100 settings,
  including computed ones and some that conflict with each other, so the
  first plans failed. We reduced `aks.tf` to the settings we actually choose
  (region, tier, node size, autoscaler 2–5, network) and re-planned until the
  plan read `2 to import, 0 to add, 0 to change, 0 to destroy`. Anything
  other than 0 changes would have meant our file disagreed with reality.
- **Two settings are ignored on purpose (`ignore_changes`):**
  - the node count, which the autoscaler changes. Otherwise every plan
    would try to reset it;
  - a field Azure leaves empty for manually managed node pools, where the
    provider would otherwise send a pointless update.
- **Guards:** `prevent_destroy` on the resource group (which also holds the
  state storage) and on the cluster. A plan that would delete or replace
  them stops with an error instead.

## Step 4: Apply the import

```sh
terraform plan -out=import.tfplan
terraform apply import.tfplan && rm import.tfplan
```

- **What it does:** saves the plan to a file, then applies exactly that file.
  The output was `Resources: 2 imported, 0 added, 0 changed, 0 destroyed`;
  nothing in Azure changed.
- **Why a saved plan:** plain `terraform apply` plans again and waits for you
  to type `yes`. Commands run from Claude Code with `!` get no keyboard input,
  so it failed with `error asking for approval: EOF`, safely and without
  changes. A saved plan needs no prompt. It applies exactly what was reviewed
  and refuses (`Saved plan is stale`) if Azure changed in between.
- **The plan file is not committed** (`*.tfplan` in `.gitignore`): it can
  contain secrets, here the cluster's kubeconfig.
- **Check:** `terraform plan` now says `No changes. Your infrastructure
  matches the configuration.`

## Step 5: An identity for GitHub Actions

`terraform/github-identity.tf` creates:

| Resource | Purpose |
|---|---|
| Managed identity `rso-github-actions` | The account GitHub workflows act as |
| 15 federated credentials | Which GitHub runs may use it (below) |
| Role *AKS Cluster User* on the cluster | CD can download a kubeconfig for `helm` |
| Role *Reader* on `rso-rg` | CI can run `terraform plan` |
| Role *Storage Blob Data Contributor* on the state account | CI can read the state and take its lock |

How it works:

1. A workflow job asks GitHub for a signed OIDC token. Its *subject* names
   the organization and repository, each with its numeric ID, and the job's
   *environment*, for example
   `repo:RSOcc26@345155378/gateway@1428425800:environment:dev`.
2. `azure/login` hands the token to Entra ID.
3. Entra ID gives back an Azure token, valid for about an hour, if that
   subject matches one of the identity's federated credentials.

The trusted subjects are `environment:dev` and `environment:prod` for each of
the 7 deploying repositories, plus `…/deploy@<id>:pull_request` for the plan
check.

- **The IDs in the subject:** our first version trusted
  `repo:RSOcc26/deploy:pull_request`, the format most guides show. The first
  CI run failed with `AADSTS700213: No matching federated identity record
  found for presented assertion subject
  'repo:RSOcc26@345155378/deploy@1430878369:pull_request'`. GitHub now
  includes the owner and repository IDs by default, so a repository that is
  deleted and recreated under the same name (by anyone) gets a new ID and is
  not trusted. The IDs are public; look one up with
  `gh api repos/RSOcc26/<repo> -q .id`. Changing the subjects was an in-place
  update (`0 to add, 15 to change, 0 to destroy`).

- **Why a managed identity:** the usual way is an *app registration* (service
  principal), but the university tenant does not allow students to create
  them. A user-assigned managed identity supports the same federated login
  and lives in our resource group.
- **Why federated credentials, not a secret:** there is nothing to leak, copy
  or rotate. A client secret or a kubeconfig in GitHub secrets is valid for
  months; these tokens expire within the hour.
- **Why per environment:** only jobs that run in a GitHub environment named
  `dev` or `prod` can deploy. Approval rules for prod are set on that
  environment, so a workflow cannot skip them.
- **Know the limit:** the cluster uses Kubernetes RBAC with local accounts,
  so the kubeconfig the *Cluster User* role downloads has full rights inside
  the cluster. Restricting it per namespace needs Entra ID integration for
  the cluster, which is not worth it for a two-person team.
- **Provider setting:** `resource_provider_registrations = "none"` in
  `versions.tf`. By default the provider registers Azure services at
  subscription level on every run. CI only has rights on the resource group,
  so every CI plan would fail with 403.

```sh
terraform plan -out=identity.tfplan     # 19 to add, 0 to change, 0 to destroy
terraform apply identity.tfplan && rm identity.tfplan
```

- **What it does:** creates the 19 resources, at no cost (managed identities
  and role assignments are free). It took about 2 minutes; a long-running
  command is moved to the background by Claude Code and still completes.
- **If a role assignment fails with `PrincipalNotFound`:** Entra ID had not
  finished replicating the new identity. Repeating plan and apply creates
  only what is missing.

## Step 6: Tell GitHub which identity to use

```sh
gh variable set AZURE_CLIENT_ID --org RSOcc26 --visibility all \
  --body "$(terraform -chdir=deploy/terraform output -raw github_client_id)"
gh variable set AZURE_TENANT_ID --org RSOcc26 --visibility all \
  --body "$(terraform -chdir=deploy/terraform output -raw github_tenant_id)"
gh variable set AZURE_SUBSCRIPTION_ID --org RSOcc26 --visibility all \
  --body "$(az account show --query id -o tsv)"
gh variable list --org RSOcc26
```

- **What it does:** stores three organization-wide Actions *variables*,
  available to every repository as `vars.AZURE_CLIENT_ID` and so on. The
  values come straight from Terraform and `az`, so nothing is copied by hand.
- **Why variables and not secrets:** they are IDs. Without a matching
  federated credential they grant nothing. As variables they stay visible in
  logs, which makes debugging easier.
- **If skipped:** `azure/login` fails with a missing `client-id`.
- **If `gh` answers 403:** the login lacks the `admin:org` scope; run
  `gh auth refresh -s admin:org`.

## What uses this

- `.github/.github/workflows/reusable-deploy.yml` (CD, [chapter 5](05-continuous-deployment.md)): signs in
  as the identity, skips with a warning if the cluster is stopped, otherwise
  runs `helm upgrade --install --rollback-on-failure` with the new image tag.
- The `Terraform` job in `deploy/.github/workflows/validate.yml`: on every PR
  checks formatting and validates; on PRs also plans against the live
  infrastructure, and the summary shows the plan result.

Applying Terraform stays a manual step on a laptop, after the PR's plan has
been reviewed.
