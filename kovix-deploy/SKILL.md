---
name: kovix-deploy
description: Prepare full AKS deployment for a Kovix monorepo project (Dockerfiles, k8s manifests, GitHub Actions CI/CD, secrets)
disable-model-invocation: true
argument-hint: "[project-name] [dev-domain] [prod-domain]"
allowed-tools: Read Grep Glob Bash Write Edit Agent
---

# Kovix AKS Deployment Setup

Set up the complete deployment infrastructure for a Kovix monorepo project on Azure Kubernetes Service.

## Arguments
- `$0` — project name (lowercase, used for namespaces, image names, etc. e.g. `anden`)
- `$1` — dev domain (e.g. `dev.anden.kovix.io`). If only one environment, this is the only domain.
- `$2` — prod domain (e.g. `anden.kovix.io`). Optional if single-environment setup.

## Prerequisites

The following CLI tools must be authenticated before running this skill:

- **Azure CLI** (`az`): logged in with `az login` to the correct tenant/subscription
- **GitHub CLI** (`gh`): logged in with `gh auth login` to the kovix-tech org
- **kubectl**: will be configured automatically via `az aks get-credentials`

## Credentials file

All infrastructure credentials must be provided in a file called `deploy-credentials.json` in the project root. **This file must be gitignored.** Read it at the start and use its values throughout.

### Schema

All top-level properties are required except `storage` (optional).
Within `clusters` and `azureCredentials`, both `dev` and `prod` are optional — include only the environments you need.

```json
{
  "subscription": "azure-subscription-id",
  "acr": {
    "name": "registry-name (without .azurecr.io)",
    "username": "acr-username",
    "password": "acr-password"
  },
  "clusters": {
    "dev": {
      "name": "aks-cluster-name",
      "resourceGroup": "aks-resource-group"
    },
    "prod": {
      "name": "aks-cluster-name",
      "resourceGroup": "aks-resource-group"
    }
  },
  "azureCredentials": {
    "dev": {
      "clientId": "",
      "clientSecret": "",
      "subscriptionId": "",
      "tenantId": ""
    },
    "prod": {
      "clientId": "",
      "clientSecret": "",
      "subscriptionId": "",
      "tenantId": ""
    }
  },
  "database": {
    "host": "pg-server.postgres.database.azure.com",
    "user": "db-user",
    "password": "db-password",
    "devName": "dev-dbname",
    "prodName": "prod-dbname"
  },
  "storage": {
    "resourceGroup": "azure-resource-group-for-storage",
    "location": "azure-region"
  }
}
```

If the file doesn't exist, create the template and ask the user to fill it in before proceeding.

Ensure `deploy-credentials.json` is in `.gitignore` before doing anything else.

### Adaptive behavior based on credentials

**Environments:** Check which keys exist in `clusters`. If only `dev` exists, create a single-environment setup (one overlay, one branch trigger, one set of secrets). If only `prod`, same. If both, create both. The GitHub Actions workflow branches, overlays, and k8s setup must match only the environments present.

**Storage:** If `storage` property is missing or null, skip Azure Storage account creation entirely. Do not create storage accounts, do not include AZURE_STORAGE_CONNECTION_STRING in k8s secrets, do not include AZURE_STORAGE_CONTAINER in configmaps.

**Database names:** If only one environment exists, use `devName` for dev-only or `prodName` for prod-only. If both, use both.

## Reference project

Clone or fetch the reference implementation from https://github.com/kovix-tech/sensation-du-temps to understand the exact structure for Dockerfiles, k8s manifests, and GitHub Actions. Match its patterns.

## Infrastructure (Kovix shared defaults)

- **Cert issuer**: `kovix-cert-issuer` (already installed in both clusters)
- **Ingress class**: `nginx`
- **Backend port**: 8080
- **Frontend port**: 3000

## Steps to execute

### 1. Read credentials and determine scope
Read `deploy-credentials.json` from project root. Determine:
- Which environments to set up (check keys in `clusters`: dev, prod, or both)
- Whether to create storage accounts (check if `storage` property exists)

### 2. Azure Storage (skip if `storage` is missing)
For each environment present in `clusters`, create a storage account in `{storage.resourceGroup}` (`{storage.location}`):
- `${project}storage{env}` (e.g. `andenstoragedev`, `andenstorageprod`)
- Each with a container called `uploads`
- Public blob access disabled, TLS 1.2
- Get connection strings for each

### 3. Next.js config
- Set `output: "standalone"` in `next.config.ts`
- Add `/health` API route returning `{ status: "ok" }`

### 4. NestJS health endpoint
- Create `HealthModule` with `GET /health` returning `{ status: "ok" }`
- Register in `AppModule`

### 5. Dockerfiles (multi-stage, npm-based)
- `Dockerfile.web` — base → builder (npm ci, turbo build) → runner (standalone Next.js, port 3000, user nextjs:1001)
- `Dockerfile.backend` — base → builder → prod-deps (npm ci --omit=dev) → runner (node dist/main.js, port 8080, user nestjs:1001)
- `.dockerignore` — exclude node_modules, .env, .next, .turbo, .git, k8s, .github, .claude, docs, tests

### 6. Kubernetes manifests (Kustomize)
Create base manifests always. Create overlays only for environments present in `clusters`.

```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── backend-deployment.yaml    (port 8080, health /health, limits 1Gi/1000m)
│   ├── backend-service.yaml       (ClusterIP 8080)
│   ├── web-deployment.yaml        (port 3000, health /health, limits 768Mi/800m, startupProbe)
│   ├── web-service.yaml           (ClusterIP 3000)
│   └── ingress.yaml               (nginx, TLS placeholder)
└── overlays/
    ├── dev/   (only if clusters.dev exists)
    │   ├── kustomization.yaml     (namespace: ${project}-dev, images tag: dev)
    │   ├── ingress-patch.yaml     (hosts: $1, api.$1, cert-issuer: kovix-cert-issuer)
    │   ├── configmap.yaml.template
    │   └── secrets.yaml.template
    └── prod/  (only if clusters.prod exists)
        ├── kustomization.yaml     (namespace: ${project}-prod, images tag: prod)
        ├── ingress-patch.yaml     (hosts: $2 or $1 if single env, api.{domain})
        ├── configmap.yaml.template
        └── secrets.yaml.template
```

ConfigMap: only include AZURE_STORAGE_CONTAINER if `storage` exists.
Secrets template: only include AZURE_STORAGE_CONNECTION_STRING if `storage` exists.

### 7. GitHub Actions (`.github/workflows/deploy.yml`)
- Trigger: push to branches matching only the environments in `clusters` + workflow_dispatch
  - If only dev: trigger on `dev` only
  - If only prod: trigger on `prod` only
  - If both: trigger on `dev` and `prod`
- Jobs: setup → build-and-push-web + build-and-push-backend (parallel) → deploy
- Deploy step with `timeout-minutes: 5`
- Image names: `${project}-web` and `${project}-backend`
- Use Kustomize to build manifests, azure/k8s-deploy@v5
- If single environment, simplify the conditionals (no need for dev/prod branching logic)

### 8. GitHub Secrets (set with `gh secret set`)
Read values from `deploy-credentials.json` and set only what's needed:

Always set:
- `ACR_NAME` ← `acr.name`
- `ACR_USERNAME` ← `acr.username`
- `ACR_PASSWORD` ← `acr.password`

If `clusters.dev` exists:
- `AKS_CLUSTER_DEV` ← `clusters.dev.name`
- `AKS_RESOURCE_GROUP_DEV` ← `clusters.dev.resourceGroup`
- `AZURE_CREDENTIALS` ← full JSON from `azureCredentials.dev` (with Azure endpoints)

If `clusters.prod` exists:
- `AKS_CLUSTER_PROD` ← `clusters.prod.name`
- `AKS_RESOURCE_GROUP_PROD` ← `clusters.prod.resourceGroup`
- `AZURE_CREDENTIALS_PROD` ← full JSON from `azureCredentials.prod` (with Azure endpoints)

The AZURE_CREDENTIALS JSON must include these extra fields beyond the 4 core ones:
```json
{
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}
```

### 9. Kubernetes cluster setup
For each environment present in `clusters`:
1. `az aks get-credentials` using `clusters.{env}` values
2. `kubectl create namespace ${project}-{env}`
3. `kubectl apply -f` the configmap template
4. `kubectl create secret generic app-secrets` with:
   - DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME from `database.*` (use devName or prodName accordingly)
   - TRUSTED_ORIGIN from the domain
   - BETTER_AUTH_SECRET (generate unique per environment)
   - AZURE_STORAGE_CONNECTION_STRING (only if `storage` exists, from storage accounts created in step 2)
   - AZURE_STORAGE_CONTAINER = "uploads" (only if `storage` exists)

## Important
- Always verify `deploy-credentials.json` is in `.gitignore` first
- Ask the user for any missing values in the credentials file
- Generate a unique BETTER_AUTH_SECRET per environment with `crypto.randomBytes(32).toString('hex')`
- Namespace pattern: `${project}-{env}`
- Only create resources for environments that exist in the credentials file
- Only create storage resources if `storage` property is present
