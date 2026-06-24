# Aegis Health — Terraform Infrastructure

Terraform configuration that provisions all Azure infrastructure required to run the Aegis Health platform. Everything the AKS cluster and its workloads depend on is managed here: networking, compute, data stores, messaging, security, and observability.

This repo is deployed first. Once `terraform apply` completes, the cluster is ready for [health-aegis/k8s-manifests](https://github.com/health-aegis/k8s-manifests) and the application pipeline in [health-aegis/health-aegis](https://github.com/health-aegis/health-aegis).

Remote state is stored in Azure Blob Storage. Authentication to Azure uses OIDC (no long-lived credentials in CI).

---

## Modules

| Module | Provisions |
|---|---|
| `resource-group` | Azure resource group |
| `networking` | VNet (`10.0.0.0/16`) with 5 subnets: `aks` (`10.0.0.0/22`), `pe` (`10.0.10.0/24`), `postgres` (`10.0.11.0/24`), `public` (`10.0.30.0/24`), `bastion` (`10.0.20.0/26`) |
| `aks` | AKS cluster with Azure CNI, AGIC add-on, Workload Identity, OIDC issuer; system and user node pools |
| `acr` | Azure Container Registry (Premium SKU) with private endpoint; grants AKS kubelet identity `AcrPull` |
| `key-vault` | Key Vault with RBAC + vault access policies; stores all application secrets |
| `cosmosdb` | CosmosDB account (MongoDB API) with private endpoint; database `aegis_db` |
| `postgres` | PostgreSQL Flexible Server in the delegated `postgres` subnet; database `aegis_imaging` |
| `storage` | Storage account with blob containers: `health-records`, `medical-images`, `uploads`, `exports` |
| `service-bus` | Service Bus Standard namespace with a `notifications` queue and a `notification-worker` auth rule |
| `doc-intelligence` | Azure Document Intelligence (Form Recognizer) S0 tier for OCR on health record documents |
| `function-app` | Azure Function App on Linux Y1 (Consumption) plan; blob trigger for processing health record uploads |
| `communication` | Azure Communication Services resource for email delivery |
| `app-gateway` | Application Gateway v2 with WAF in the `public` subnet; used by AKS AGIC as the external ingress |
| `bastion` | Azure Bastion for secure SSH/RDP access to AKS nodes without a public IP |
| `monitoring` | Log Analytics workspace + Azure Monitor; AKS OMS agent sends container logs here |
| `workload-identity` | User-assigned managed identity + federated credential bound to `system:serviceaccount:aegis:aegis-workload-identity`; used by pods to access Key Vault without secrets |

---

## Architecture

```
Public subnet
  └── Application Gateway v2 (WAF)
        └── AGIC controls this from inside AKS

AKS subnet (10.0.0.0/22)  [Azure CNI — pods get VNet IPs]
  ├── System node pool     (kube-system workloads)
  └── User node pool       (aegis application workloads)
        └── Workload Identity → Key Vault (via federated credential)

PE subnet (10.0.10.0/24)   [Private endpoints]
  ├── CosmosDB private endpoint
  ├── ACR private endpoint
  └── Storage private endpoint

Postgres subnet (10.0.11.0/24)  [Delegated to PostgreSQL Flexible]
  └── PostgreSQL Flexible Server

Bastion subnet (10.0.20.0/26)
  └── Azure Bastion
```

ACR, CosmosDB, and Storage are only reachable from within the VNet through their private endpoints. The Function App (Consumption plan) uses public access because Consumption plan does not support VNet injection for inbound traffic.

---

## Prerequisites

Tools required locally and in CI:

- **Terraform** 1.7.5+ (`tfenv` recommended for version pinning)
- **Azure CLI** 2.55+ — for initial setup and credential verification
- **`yq`** 4+ — used by the GitOps sync job in CI to patch Helm values
- An Azure subscription with **Contributor** and **User Access Administrator** roles for the deploying identity (role assignments are created by Terraform)
- A storage account for Terraform remote state (created manually before first `init`)

### Bootstrap the remote state storage

Do this once before running `terraform init`:

```bash
az group create --name rg-tfstate --location centralindia

az storage account create \
  --name <unique-storage-name> \
  --resource-group rg-tfstate \
  --location centralindia \
  --sku Standard_LRS \
  --allow-blob-public-access false

az storage container create \
  --name tfstate \
  --account-name <unique-storage-name>
```

---

## Setup

### 1. Clone and fill in variables

```bash
git clone https://github.com/health-aegis/terraform-aegis.git
cd terraform-aegis
```

Edit `environments/test/terraform.tfvars` (dev) or `environments/prod/terraform.tfvars`. Two fields that must be set before the first apply:

```hcl
deployer_object_id = ""  # az ad signed-in-user show --query id -o tsv
deployer_ip        = ""  # curl ifconfig.me  — your public IP for Key Vault ACL
```

Do not put secrets in `.tfvars`. Set them as environment variables instead:

```bash
export TF_VAR_postgres_admin_password="YourStrongP@ssw0rd!"
export TF_VAR_jwt_secret="your-jwt-secret"
export TF_VAR_gemini_api_key="your-gemini-key"
export TF_VAR_azure_ai_key="your-doc-intelligence-key"
export TF_VAR_azure_ai_endpoint="https://your-docai.cognitiveservices.azure.com/"
```

### 2. Configure the backend

Create `environments/test/backend.hcl`:

```hcl
resource_group_name  = "rg-tfstate"
storage_account_name = "<unique-storage-name>"
container_name       = "tfstate"
key                  = "aegis-test.tfstate"
```

### 3. Init and plan

```bash
terraform init -backend-config=environments/test/backend.hcl -reconfigure

terraform plan \
  -var-file=environments/test/terraform.tfvars \
  -out=tfplan.binary
```

Review the plan output. Check that private endpoints, role assignments, and federated credentials look correct before applying.

### 4. Apply

```bash
terraform apply tfplan.binary
```

A full apply from scratch takes roughly 20-30 minutes; AKS and Application Gateway are the slowest resources.

### 5. Connect kubectl

```bash
az aks get-credentials \
  --resource-group rg-aegis \
  --name <aks-cluster-name> \
  --admin
```

---

## CI/CD Workflow

The `terraform-apply.yml` workflow wraps the reusable `terraform-core.yml`.

**On push to `main`** (when `.tf`, `.tfvars`, or `.hcl` files change): runs plan only against the `test` environment and posts the plan output to the Actions summary. No infrastructure is changed.

**On `workflow_dispatch`**: you choose the environment (`test` or `prod`) and the action (`plan` or `apply`). Selecting `apply` in the dispatch form is the approval step.

The pipeline has three stages in `terraform-core.yml`:

1. **plan** — `terraform validate`, `terraform fmt -check`, `terraform plan`; uploads the binary plan as an artifact and posts a summary to the Actions run.
2. **apply** — downloads the plan artifact, runs `terraform apply`. Captures `workload_identity_client_id`, `key_vault_name`, and `tenant_id` as job outputs.
3. **sync-gitops** — checks out `k8s-manifests`, patches `environments/<env>/values.yaml` with the real Terraform outputs (`clientId`, `tenantId`, `keyVault.name`), and commits. This keeps the Helm deployment values in sync with the actual infrastructure after a destroy-and-recreate.

### Required GitHub Actions secrets

| Secret | Description |
|---|---|
| `AZURE_CLIENT_ID` | Client ID of the federated identity used for OIDC login |
| `AZURE_TENANT_ID` | Azure tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID |
| `TF_VAR_POSTGRES_ADMIN_PASSWORD` | PostgreSQL admin password |
| `TF_VAR_JWT_SECRET` | JWT signing secret (written to Key Vault by Terraform) |
| `TF_VAR_AZURE_AI_KEY` | Document Intelligence API key |
| `TF_VAR_AZURE_AI_ENDPOINT` | Document Intelligence endpoint |
| `TF_VAR_GEMINI_API_KEY` | Google Gemini API key |
| `GITOPS_PAT` | GitHub PAT with repo write on `health-aegis/k8s-manifests` for the GitOps sync step |

---

## Repo Structure

```
.
├── .github/
│   └── workflows/
│       ├── terraform-apply.yml   # Caller: push → plan, dispatch → plan or apply
│       ├── terraform-core.yml    # Reusable: validate / plan / apply / sync-gitops
│       └── terraform-destroy.yml # Manual destroy workflow
├── modules/
│   ├── resource-group/
│   ├── networking/
│   ├── aks/
│   ├── acr/
│   ├── key-vault/
│   ├── cosmosdb/
│   ├── postgres/
│   ├── storage/
│   ├── service-bus/
│   ├── doc-intelligence/
│   ├── function-app/
│   ├── communication/
│   ├── app-gateway/
│   ├── bastion/
│   ├── monitoring/
│   └── workload-identity/
├── environments/
│   ├── test/
│   │   ├── backend.hcl           # Remote state config for test/dev
│   │   └── terraform.tfvars      # Variable values for test/dev
│   └── prod/
│       ├── backend.hcl
│       └── terraform.tfvars
├── main.tf                       # Root module — calls all child modules
├── variables.tf                  # Variable declarations
├── outputs.tf                    # Outputs used by the GitOps sync step
├── terraform.tfvars              # Default variable values (non-secret)
└── .gitignore                    # Excludes .terraform/, *.tfstate, *.tfvars secrets
```

---

## Key Design Decisions

**Azure CNI over kubenet.** Pods get IPs from the AKS subnet directly, which allows private endpoint DNS resolution to work without extra configuration. Kubenet uses NAT for pod traffic, which breaks private endpoint reachability from pods.

**Private endpoints for PaaS services.** CosmosDB, ACR, and Storage are only accessible within the VNet. The Function App is an exception; Consumption plan does not support VNet integration for inbound connections, so it uses public access.

**Workload Identity over pod-managed identity.** Each pod annotates its ServiceAccount with the managed identity client ID. The CSI driver uses this to authenticate to Key Vault when mounting secrets. No node-level identity is shared between pods.

**AGIC instead of nginx ingress.** Application Gateway handles TLS termination and WAF inspection before traffic reaches the cluster. AGIC is the in-cluster controller that programs the gateway based on Kubernetes Ingress resources. This removes nginx from the data path entirely.

**Destroy workflow is separate.** `terraform-destroy.yml` is a manual-only dispatch that requires explicitly typing the environment name to prevent accidental destruction.

---

## Outputs

After apply, the following outputs are available and are used by the GitOps sync step:

```bash
terraform output workload_identity_client_id  # Annotated on the AKS ServiceAccount
terraform output key_vault_name               # Set in k8s-manifests values.yaml
terraform output tenant_id                    # Set in k8s-manifests values.yaml
```
