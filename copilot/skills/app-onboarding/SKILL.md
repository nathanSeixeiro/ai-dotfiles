---
name: app-onboarding
description: Guide for containerizing and deploying a new application on MyOrg/MyOrg Kubernetes (EKS/AKS). Use when the request involves a new app, containerize, new namespace, eks deploy, aks deploy, gitops setup, pipeline setup, or creating a Dockerfile.
---

# App Onboarding — Containerization and Deploy

## Repositories

| Repo | Use |
|------|-----|
| `my-org/k8s-gitops` | GitOps manifests (Kustomize) — K8s resources |
| `my-org/k8s-infra` | Terraform — clusters, Key Vaults, IAM |
| `my-org/devops-workflows` | Reusable workflows (CI/CD) |

## Checklist — New App on Kubernetes

### 1. Project (application repo)

- [ ] Create branch `feat/containerize` from `develop`
- [ ] Create `Dockerfile` (multi-stage)
- [ ] Create `.github/workflows/<app>-ci-cd.yml`
- [ ] Push and test pipeline

### 2. Infra (k8s-infra)

- [ ] Create Key Vault: `Aks/terraform/keyvaults/environments/<env>/kv-<app-name>.tf`
- [ ] After the pipeline runs, set real values via `az keyvault secret set`

### 3. GitOps (k8s-gitops)

#### EKS (path: `Eks/kubernetes/...`)
- [ ] Create namespace overlay: `Eks/kubernetes/overlays/<env>/ns-<app>-<env>/`
- [ ] Create app manifests: `deployment.yaml`, `service.yaml`, `configmap.yaml`, `external-secret.yaml`, `kustomization.yaml`
- [ ] Create `pdb.yaml` (PodDisruptionBudget) — **required in prd/stg** (min/hml optional)
- [ ] Create scheduled-scaling (KEDA ScaledObject) — see "Autoscaling + PDB Pattern" below
- [ ] Create `ClusterSecretStore`: `Eks/kubernetes/base/external-secrets/<env>/cluster-secret-store-<app>.yaml`
- [ ] **Register** the ClusterSecretStore in `Eks/kubernetes/base/external-secrets/<env>/kustomization.yaml`
- [ ] Create ArgoCD `AppProject`: `Eks/kubernetes/argocd/<env>/projects/<app>-<env>.yaml`
- [ ] In the AppProject, include `policy/PodDisruptionBudget` in `namespaceResourceWhitelist` (otherwise ArgoCD blocks the PDB)
- [ ] **Register** it in `Eks/kubernetes/argocd/<env>/projects/kustomization.yaml`
- [ ] Create ArgoCD `Application`: `Eks/kubernetes/argocd/<env>/applications/<app>-<env>.yaml`
- [ ] **Register** it in `Eks/kubernetes/argocd/<env>/applications/kustomization.yaml`
- [ ] Add the namespace to the overlay root `kustomization.yaml`

#### AKS (path: `kubernetes/...` — no prefix)
- [ ] Create namespace overlay: `kubernetes/overlays/<env>/ns-<app>-<env>/`
- [ ] Create app manifests: `deployment.yaml`, `service.yaml`, `configmap.yaml`, `external-secret.yaml`, `kustomization.yaml`
- [ ] Create `pdb.yaml` (PodDisruptionBudget) — **required in prd/stg**
- [ ] Create scheduled-scaling (KEDA ScaledObject) — see "Autoscaling + PDB Pattern" below
- [ ] Create `ClusterSecretStore`: `kubernetes/base/external-secrets/<env>/cluster-secret-store-<app>.yaml`
- [ ] Create ArgoCD `AppProject` and `Application` (include `policy/PodDisruptionBudget` in the AppProject whitelist)
- [ ] Add the namespace to the overlay root `kustomization.yaml`

## Secrets — Both Clouds use Azure Key Vault

### Responsibility separation (MUST remember)

Secrets involve TWO different repos — both are needed when creating a new environment:

| Resource | Repo | Path | Created via |
|----------|------|------|-------------|
| **Key Vault** (Azure resource) | `k8s-infra` | `Aks/terraform/keyvaults/environments/<env>/kv-<app>.tf` | Terraform (plan/apply via CI/CD) |
| **ClusterSecretStore** (points to the KV) | `k8s-gitops` | `Eks/kubernetes/base/external-secrets/<env>/` (or `kubernetes/...` on AKS) | GitOps / ArgoCD |
| **ExternalSecret** (reads KV -> K8s Secret) | `k8s-gitops` | inside the app overlay | GitOps / ArgoCD |

- The KV (Terraform) provisions the vault + RBAC (ESO reader + admin) + PLACEHOLDER secrets (`ignore_changes`).
- The ClusterSecretStore/ExternalSecret (gitops) only point and read — they assume the KV already exists.
- **When creating a new environment (e.g.: prd), open a PR in infra (KV) AND in gitops (CSS+ExternalSecret).** Forgetting the KV is a common mistake.

### Naming pattern

- KV name: `kv-<env-prefix>-<app-short-name>` (max 24 chars)
  - `d` = dev, `h` = hml, `p` = prd
- Secret names in the KV: use `--` as separator (Azure does not accept `__`)
  - E.g.: `ConnectionStrings--db`, `ApiConfig--ApiKey`
- In K8s (envvar): `__` as separator
  - E.g.: `ConnectionStrings__db`, `ApiConfig__ApiKey`

### Set values in the Key Vault

```bash
az keyvault secret set \
  --vault-name "kv-d-<app-short-name>" \
  --name "<SecretName--Property>" \
  --value "<real-value>"
```

### ExternalSecret maps KV -> K8s Secret

```yaml
spec:
  secretStoreRef:
    name: azure-kv-<app-name>
    kind: ClusterSecretStore
  data:
    - secretKey: ConnectionStrings__db    # name in K8s
      remoteRef:
        key: ConnectionStrings--db        # name in KV
```

## Pattern — Static Sites (HTML/CSS/JS)

Canonical reference: repo `my-org/hotsite-a`.

Every static site follows this pattern:

### Repo structure
```
<repo>/
├── .github/workflows/ci-cd.yml
├── Dockerfile
└── Site/            # all static content (index.html, assets, etc)
```

### Dockerfile (httpd Apache, port 80)
```dockerfile
FROM httpd:2.4
COPY Site/. /usr/local/apache2/htdocs/
EXPOSE 80
```
> Static site (httpd/nginx) uses **port 80** — see "Port rule per stack" below.

### Pipeline (EKS GitOps)
- Based on the EKS model below, simplified version (no build/sonar — a static site does not compile)
- Jobs: `docker` -> `deploy-gitops` -> `argocd-sync`
- `deploy-gitops` passes `environment: 'review'` as an **input** of the reusable `update-image-tag.yml` (production gate; it is NOT a job key, do NOT use `'prd'`)
- Aggregator namespace in ArgoCD (e.g.: `hotsite-a-prd`, `corp-hotsite-*`) -> use `resource_prefix: <component>` in `argocd-sync-deployment`

```yaml
name: <Site Name> - CI/CD - EKS

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  docker:
    name: Docker Build - [${{ github.ref_name }}]
    uses: my-org/devops-workflows/.github/workflows/docker-build-aws.yml@main
    with:
      dockerfile: "Dockerfile"
      image: "<image-name>"
      tag: ${{ github.run_number }}
      runs_on: ubuntu-latest
      aws_region: "us-east-1"
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.PRD_AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.PRD_AWS_SECRET_ACCESS_KEY }}

  deploy-gitops:
    if: ${{ always() && needs.docker.result == 'success' && github.ref_name == 'main' }}
    name: GitOps Deploy - [${{ github.ref_name }}]
    needs: docker
    uses: my-org/devops-workflows/.github/workflows/update-image-tag.yml@main
    with:
      app: '<app-name>'
      environment: 'review'
      image: '333333333333.dkr.ecr.us-east-1.amazonaws.com/<image>'
      tag: ${{ github.run_number }}
      path_prefix: "Eks/"
      kustomization_path: "kubernetes/overlays/prd/ns-<ns>-prd/<component>"
    secrets: inherit

  argocd-sync:
    name: "ArgoCD Sync - [${{ github.ref_name }}]"
    if: ${{ !cancelled() && needs.deploy-gitops.result == 'success' }}
    needs: deploy-gitops
    uses: my-org/devops-workflows/.github/workflows/argocd-sync-deployment.yml@main
    with:
      argocd_app_name: '<ns>-prd'
      resource_prefix: <component>
    secrets:
      ARGOCD_SERVER: ${{ secrets.ARGOCD_SERVER_EKS_PRD }}
      ARGOCD_USERNAME: ${{ secrets.ARGOCD_USERNAME_EKS_PRD }}
      ARGOCD_PASSWORD: ${{ secrets.ARGOCD_PASSWORD_EKS_PRD }}
```

### Static site rules
- No secrets/ExternalSecret by default (a static site has no sensitive config). Only add if truly needed.
- Deployment uses the httpd image on 80, TCP probes on 80.
- `environment: 'review'` required as the INPUT of `deploy-gitops` for pipelines going to prd.

### ATTENTION — `environment` is an INPUT of the reusable, NOT a job gate

- In `deploy-gitops`, `environment: 'review'` goes INSIDE `with:` (it is an input of the reusable `update-image-tag.yml`).
- Do **NOT** put an `environment:` key at the job level (a job using `uses:` of a reusable does NOT accept it — breaks with: "Unexpected value 'uses'").
- Do **NOT** create a separate `approval` job.
- The value is **`review`** (not `prd`). Working reference: `hotsite-a`.
- **GENERAL RULE (every pipeline, EKS and AKS):** in the `environment` of `deploy-gitops`, the production branch (`main`) ALWAYS resolves to `'review'`, NEVER `'prd'`. `review` is the production gate of the reusable. E.g.: `environment: ${{ github.ref_name == 'develop' && 'dvp' || github.ref_name == 'homolog' && 'hml' || 'review' }}`. This holds even when `image`/`kustomization_path` point to prd.

## Autoscaling + PDB Pattern

Apply to EVERY new app. Difference per environment:

| Environment | ScaledObject (KEDA) | PDB | base replicas |
|-------------|---------------------|-----|---------------|
| **prd / stg** | CPU+mem Utilization, `min 2 / max 4` | `minAvailable: 1` (required) | 2 |
| **dev / hml** | cron (on/off business hours) OR CPU, `min 0-1` | optional | 1 |

### Rules
- **prd/stg does NOT use cron.** It uses CPU/mem autoscale `min 2 / max 4` (HA + scale under load). Same as other prd apps.
- **One ScaledObject per Deployment.** Never two (cron + cpu) on the same target — they generate 2 HPAs that fight.
- dev/hml may keep cron for cost saving (min 0/max 1, on 7am off 9pm).
- PDB `minAvailable: 1` whenever there are >= 2 replicas (prd/stg).
- `requests.cpu` must be realistic (e.g.: 500m) — HPA computes Utilization over the request.

### ScaledObject prd/stg (CPU + mem)
File: `<overlay>/scheduled-scaling/scaledobject-<app>-<env>.yaml`
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: <app>-<env>
  namespace: ns-<app>-<env>
spec:
  scaleTargetRef:
    name: <app>-<env>
  minReplicaCount: 2
  maxReplicaCount: 4
  pollingInterval: 30
  cooldownPeriod: 300
  triggers:
    - type: cpu
      metricType: Utilization
      metadata:
        value: "70"
    - type: memory
      metricType: Utilization
      metadata:
        value: "70"
```

### PDB (prd/stg)
File: `<overlay>/<app>-<env>/pdb.yaml` — register it in the app `kustomization.yaml`.
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: <app>-<env>
  namespace: ns-<app>-<env>
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: <app>-<env>
      app.kubernetes.io/instance: <app>-<env>
```

### AppProject — allow the PDB (COMMON TRAP)
ArgoCD blocks sync if the kind is not in `namespaceResourceWhitelist`. Typical error:
```
resource policy:PodDisruptionBudget is not permitted in project <app>-<env>
```
ALWAYS include in the AppProject:
```yaml
  namespaceResourceWhitelist:
    # ... other kinds ...
    - group: keda.sh
      kind: ScaledObject
    - group: keda.sh
      kind: TriggerAuthentication
    - group: policy
      kind: PodDisruptionBudget
```

### Resources (deployment) — prd baseline
```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "1"        # adjust to the app weight (heavy reporting/processing)
    memory: 1Gi
```

## OpenTelemetry Instrumentation (Splunk O11y)

Instrument apps via the OTel Operator (auto-instrumentation). It goes in the **deployment**, not in the ConfigMap.

### Where to put it — always the deployment
- **Annotations** (pod template) — the OTel Operator only reads pod annotations to inject auto-instrumentation. A ConfigMap does not work.
- **OTLP envs** — they use `fieldRef` (hostIP) and `$(VAR)` interpolation, features exclusive to `env` in the container. A ConfigMap does NOT support `fieldRef` nor interpolate another env. Keep the 3 together in the deployment.
- ConfigMap = business config (Issuer, LogLevel). Platform instrumentation = deployment.

### .NET — annotations + envs
```yaml
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-dotnet: "splunk-o11y/splunk-otel-collector"
        instrumentation.opentelemetry.io/otel-dotnet-auto-runtime: "linux-x64"
    spec:
      containers:
        - name: <app>
          env:
            # OTLP straight to the hostPort of splunk-otel-collector-agent (DaemonSet).
            # Needed because dnsPolicy: None points to corporate DNS,
            # which does not resolve *.svc.cluster.local. Without this the export fails.
            # The OTel Operator does not override an env already declared in the container.
            - name: SPLUNK_OTEL_AGENT
              valueFrom:
                fieldRef:
                  fieldPath: status.hostIP
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://$(SPLUNK_OTEL_AGENT):4318"
            - name: OTEL_EXPORTER_OTLP_PROTOCOL
              value: "http/protobuf"
```

### Rules
- Apps with `dnsPolicy: None` (default in the deployments here) **need** `SPLUNK_OTEL_AGENT` via hostIP — otherwise the OTLP export fails (it does not resolve the collector Service).
- `otel-dotnet-auto-runtime`: `linux-x64` (standard .NET images) or `linux-musl-x64` (Alpine).
- Apply per environment as needed (e.g.: prd only, or prd+hml).

## Port rule per stack

The container/EXPOSE port and the probe port depend on the stack:

| Stack | Port | Reason |
|-------|------|--------|
| **.NET 8+** (aspnet 8/9+) | **8080** | Image runs as non-root by default; Kestrel uses 8080 (`ASPNETCORE_URLS=http://+:8080`) |
| **.NET < 8** (aspnet 6/7 and earlier) | **80** | Old default (`ASPNETCORE_URLS=http://+:80` or implicit) |
| **Static site** (httpd/nginx) | **80** | Web server default port; do not change |
| **Other stacks** (node, php, etc) | **80** (default) | Unless the image/base requires another |

Practical rule: **8080 is specific to .NET 8+**. Everything else (static site, old .NET, other stacks) normally uses **80**. Deployment/Service/probes must point to the real container port.

## Dockerfile template (.NET)

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:<version> AS base
WORKDIR /app
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080

FROM mcr.microsoft.com/dotnet/sdk:<version> AS build
ARG AZURE_ARTIFACTS_TOKEN
WORKDIR /src

# If there is a private nuget.config
COPY nuget.config .
RUN if [ -n "$AZURE_ARTIFACTS_TOKEN" ]; then \
      sed -i "s/%GITHUB_TOKEN%/$AZURE_ARTIFACTS_TOKEN/g" nuget.config; \
    fi

COPY <Project>/<Project>.csproj <Project>/
RUN dotnet restore "<Project>/<Project>.csproj"

COPY <Project>/ <Project>/
RUN dotnet build "<Project>/<Project>.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "<Project>/<Project>.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "<Project>.dll"]
```

## Pipeline Model — AWS (EKS)

### Monorepo (multiple projects in the same repo)
- Create **one pipeline per project**: `api-ci-cd.yml`, `web-ci-cd.yml`
- Use cross `paths-ignore` (api ignores web and vice versa)
- Inline build step (not reusable) with `dotnet restore` + `dotnet build` + SonarQube
- Each Dockerfile lives inside its project: `Pes.WebAPI/Dockerfile`, `Pes.WebAppGestao/Dockerfile`
- Reference: repo `app-d-basic`

```yaml
name: <App Name> - CI/CD

on:
  push:
    branches: [develop, homolog, main]
    paths:
      - '<ProjectFolder>/**'
      - '.github/workflows/<workflow>.yml'
  workflow_dispatch:

permissions:
  contents: read

jobs:
  build:
    name: Build & Sonar - [${{ github.ref_name }}]
    uses: my-org/devops-workflows/.github/workflows/build-sonar.yml@main
    with:
      project_path: "<ProjectFolder>/<Project>.csproj"
      dotnet_version: "<version>.x"
      sonar_project_key: "ORG_<repo-name>"
      sonar_project_name: "<repo-name>"
      run_tests: false
      run_sonar: ${{ github.ref_name == 'develop' }}
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
      NUGET_AUTH_TOKEN: ${{ secrets.GH_ARTIFACTS_TOKEN }}  # optional — only if private nuget feed

  transform:
    if: ${{ github.ref_name != 'develop' && github.ref_name != 'homolog' }}
    name: Transform appsettings.json
    uses: my-org/devops-workflows/.github/workflows/transform-appsettings.yml@main
    needs: build
    with:
      appsettings_path: "<ProjectFolder>/appsettings.json"
      environment_name: ${{ github.ref_name == 'main' && 'production' || 'staging' }}
    secrets:
      APPSETTINGS_SECRETS: ${{ toJSON(secrets) }}

  docker:
    if: |
      always() &&
      needs.build.result == 'success' &&
      (needs.transform.result == 'success' || needs.transform.result == 'skipped')
    name: Docker Build - [${{ github.ref_name }}]
    needs: [build, transform]
    uses: my-org/devops-workflows/.github/workflows/docker-build-aws.yml@main
    with:
      dockerfile: "<ProjectFolder>/Dockerfile"
      image: <ecr-image-name>
      tag: ${{ github.run_number }}
      app_folder: ${{ (github.ref_name != 'develop' && github.ref_name != 'homolog') && '<ProjectFolder>' || '' }}
      runs_on: ubuntu-latest
      aws_region: "us-east-1"
    secrets:
      AWS_ACCESS_KEY_ID: ${{ github.ref_name == 'main' && secrets.PRD_AWS_ACCESS_KEY_ID || github.ref_name == 'homolog' && secrets.HML_AWS_ACCESS_KEY_ID || secrets.DEV_AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ github.ref_name == 'main' && secrets.PRD_AWS_SECRET_ACCESS_KEY || github.ref_name == 'homolog' && secrets.HML_AWS_SECRET_ACCESS_KEY || secrets.DEV_AWS_SECRET_ACCESS_KEY }}

  deploy-gitops:
    if: |
      always() &&
      needs.docker.result == 'success' &&
      (github.ref_name == 'develop' || github.ref_name == 'homolog' || github.ref_name == 'main')
    name: GitOps Deploy - [${{ github.ref_name }}]
    needs: docker
    uses: my-org/devops-workflows/.github/workflows/update-image-tag.yml@main
    with:
      app: '<app-name>'
      environment: ${{ github.ref_name == 'develop' && 'dvp' || github.ref_name == 'homolog' && 'hml' || 'review' }}
      image: ${{ github.ref_name == 'develop' && '111111111111.dkr.ecr.us-east-1.amazonaws.com/<image>' || github.ref_name == 'homolog' && '222222222222.dkr.ecr.us-east-1.amazonaws.com/<image>' || '333333333333.dkr.ecr.us-east-1.amazonaws.com/<image>' }}
      tag: ${{ github.run_number }}
      path_prefix: "Eks/"
      kustomization_path: ${{ github.ref_name == 'develop' && 'kubernetes/overlays/dev/ns-<app>-dev/<app-component>' || github.ref_name == 'homolog' && 'kubernetes/overlays/hml/ns-<app>-hml/<app-component>-hml' || 'kubernetes/overlays/prd/ns-<app>-prd/<app-component>' }}
    secrets: inherit

  argocd-sync:
    name: "ArgoCD Sync - [${{ github.ref_name }}]"
    if: ${{ !cancelled() && needs.deploy-gitops.result == 'success' }}
    needs: deploy-gitops
    uses: my-org/devops-workflows/.github/workflows/argocd-sync-deployment.yml@main
    with:
      argocd_app_name: ${{ github.ref_name == 'develop' && '<app>-dev' || github.ref_name == 'homolog' && '<app>-hml' || '<app>-prd' }}
      resource_prefix: <app-component>
    secrets:
      ARGOCD_SERVER: ${{ github.ref_name == 'develop' && secrets.ARGOCD_SERVER_EKS_DEV || github.ref_name == 'homolog' && secrets.ARGOCD_SERVER_EKS_HML || secrets.ARGOCD_SERVER_EKS_PRD }}
      ARGOCD_USERNAME: ${{ github.ref_name == 'develop' && secrets.ARGOCD_USERNAME_EKS_DEV || github.ref_name == 'homolog' && secrets.ARGOCD_USERNAME_EKS_HML || secrets.ARGOCD_USERNAME_EKS_PRD }}
      ARGOCD_PASSWORD: ${{ github.ref_name == 'develop' && secrets.ARGOCD_PASSWORD_EKS_DEV || github.ref_name == 'homolog' && secrets.ARGOCD_PASSWORD_EKS_HML || secrets.ARGOCD_PASSWORD_EKS_PRD }}
```

## Pipeline Model — Azure (AKS)

```yaml
name: <App Name> - CI/CD

on:
  push:
    branches: [develop, homolog, staging, main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  build:
    name: Build & Sonar - [${{ github.ref_name }}]
    uses: my-org/devops-workflows/.github/workflows/build-sonar.yml@main
    with:
      project_path: "<ProjectFolder>/<Project>.csproj"
      dotnet_version: "<version>.x"
      sonar_project_key: "ORG_<repo-name>"
      sonar_project_name: "<repo-name>"
      run_tests: false
      run_sonar: ${{ github.ref_name == 'develop' }}
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
      NUGET_AUTH_TOKEN: ${{ secrets.GH_ARTIFACTS_TOKEN }}  # optional — only if private nuget feed

  transform:
    if: ${{ github.ref_name == 'main' || github.ref_name == 'staging' }}
    name: Transform appsettings.json
    uses: my-org/devops-workflows/.github/workflows/transform-appsettings.yml@main
    needs: build
    with:
      appsettings_path: "<ProjectFolder>/appsettings.json"
      environment_name: ${{ github.ref_name == 'main' && 'production' || 'staging' }}
    secrets:
      APPSETTINGS_SECRETS: ${{ toJSON(secrets) }}

  docker:
    if: |
      always() &&
      needs.build.result == 'success' &&
      (needs.transform.result == 'success' || needs.transform.result == 'skipped')
    name: Docker Build - [${{ github.ref_name }}]
    needs: [build, transform]
    uses: my-org/devops-workflows/.github/workflows/docker-build.yml@main
    with:
      dockerfile: "Dockerfile"
      image: ${{ github.ref_name == 'staging' && '<image-name>-staging' || '<image-name>' }}
      runs_on: ubuntu-latest
      app_folder: ${{ (github.ref_name == 'main' || github.ref_name == 'staging') && '<ProjectFolder>/' || '' }}
    secrets:
      REGISTRY_URL: ${{ (github.ref_name == 'main' || github.ref_name == 'staging') && secrets.REGISTRY_URL_PRD || github.ref_name == 'homolog' && secrets.REGISTRY_URL_HML || secrets.REGISTRY_URL_DEV }}
      REGISTRY_LOGIN: ${{ (github.ref_name == 'main' || github.ref_name == 'staging') && secrets.REGISTRY_LOGIN_PRD || github.ref_name == 'homolog' && secrets.REGISTRY_LOGIN_HML || secrets.REGISTRY_LOGIN_DEV }}
      REGISTRY_PASSWORD: ${{ (github.ref_name == 'main' || github.ref_name == 'staging') && secrets.REGISTRY_PASSWORD_PRD || github.ref_name == 'homolog' && secrets.REGISTRY_PASSWORD_HML || secrets.REGISTRY_PASSWORD_DEV }}
      AZURE_ARTIFACTS_TOKEN: ${{ secrets.AZURE_ARTIFACTS_TOKEN }}

  deploy-gitops:
    if: |
      always() &&
      needs.docker.result == 'success' &&
      (github.ref_name == 'develop' || github.ref_name == 'homolog' || github.ref_name == 'staging' || github.ref_name == 'main')
    name: GitOps Deploy - [${{ github.ref_name }}]
    needs: docker
    uses: my-org/devops-workflows/.github/workflows/update-image-tag.yml@main
    with:
      app: <app-name>
      environment: ${{ github.ref_name == 'develop' && 'dvp' || github.ref_name == 'homolog' && 'hml' || github.ref_name == 'staging' && 'stg' || 'review' }}
      image: ${{ github.ref_name == 'develop' && 'acrdev.azurecr.io/<image>' || github.ref_name == 'homolog' && 'acrhml.azurecr.io/<image>' || 'acrprd.azurecr.io/<image>' }}
      tag: ${{ github.run_number }}
      kustomization_path: ${{ github.ref_name == 'develop' && 'kubernetes/overlays/dev/ns-<app>-dev/<component>' || github.ref_name == 'homolog' && 'kubernetes/overlays/hml/ns-<app>-hml/<component>' || github.ref_name == 'staging' && 'kubernetes/overlays/stg/ns-<app>-stg/<component>' || 'kubernetes/overlays/prd/ns-<app>-prd/<component>' }}
    secrets: inherit

  argocd-sync:
    name: "ArgoCD Sync - [${{ github.ref_name }}]"
    if: ${{ !cancelled() && needs.deploy-gitops.result == 'success' }}
    needs: deploy-gitops
    uses: my-org/devops-workflows/.github/workflows/argocd-sync-deployment.yml@main
    with:
      argocd_app_name: ${{ github.ref_name == 'develop' && '<app>-dev' || github.ref_name == 'homolog' && '<app>-hml' || github.ref_name == 'staging' && '<app>-stg' || '<app>-prd' }}
      resource_prefix: <component>
    secrets:
      ARGOCD_SERVER: ${{ github.ref_name == 'develop' && secrets.ARGOCD_SERVER_AKS_DEV || github.ref_name == 'homolog' && secrets.ARGOCD_SERVER_AKS_HML || secrets.ARGOCD_SERVER_AKS_PRD }}
      ARGOCD_USERNAME: ${{ github.ref_name == 'develop' && secrets.ARGOCD_USERNAME_AKS_DEV || github.ref_name == 'homolog' && secrets.ARGOCD_USERNAME_AKS_HML || secrets.ARGOCD_USERNAME_AKS_PRD }}
      ARGOCD_PASSWORD: ${{ github.ref_name == 'develop' && secrets.ARGOCD_PASSWORD_AKS_DEV || github.ref_name == 'homolog' && secrets.ARGOCD_PASSWORD_AKS_HML || secrets.ARGOCD_PASSWORD_AKS_PRD }}
```

## AWS Account IDs (ECR)

| Environment | Account ID | ECR |
|-------------|-----------|-----|
| dev | 111111111111 | 111111111111.dkr.ecr.us-east-1.amazonaws.com |
| hml | 222222222222 | 222222222222.dkr.ecr.us-east-1.amazonaws.com |
| prd | (use PRD_AWS_ECR secret) | — |

## Azure Container Registries (ACR)

| Environment | Registry |
|-------------|----------|
| dev | acrdev.azurecr.io |
| hml | acrhml.azurecr.io |
| prd | acrprd.azurecr.io |

## EKS Clusters

| Environment | Cluster | Region |
|-------------|---------|--------|
| dev | eks-dev | us-east-1 |
| hml | eks-hml | us-east-1 |
| prd | eks-prod | us-east-1 |

## AKS Clusters

| Environment | Cluster | Region |
|-------------|---------|--------|
| dev | aks-dvp | brazilsouth |
| hml | aks-hml | brazilsouth |
| prd | aks-pro | brazilsouth |

## Transform appsettings

- **develop/homolog**: does NOT transform — uses `appsettings.json` directly + KV secrets via ExternalSecret
- **staging/main**: transforms appsettings before the docker build (replaces sensitive values to bake into the image)
- `app_folder` in docker-build controls whether it copies the transformed folder or the source directly

## Rules

- **All environments** (develop, homolog, staging, main/prd): deploy via **GitOps** (update-image-tag -> ArgoCD sync)
- Direct deploy via kubectl/helm (deploy.yml / deploy-eks.yml) is **deprecated** — do not use in a new pipeline
- Sonar runs **on develop only**
- No transform by default. Transform only if the app bakes appsettings (legacy) — avoid in new apps

### EKS vs AKS deploy difference

| | EKS | AKS |
|--|-----|-----|
| dev/hml | GitOps | GitOps |
| stg/prd | **GitOps** (all environments) | **GitOps** (all environments) |
| path_prefix in update-image-tag | `"Eks/"` | `"Aks/"` (default in the reusable, no need to pass) |
| argocd-sync workflow | `argocd-sync-deployment.yml` | `argocd-sync-deployment.yml` |
| Alternative branch | — | may be `ci/<env>` during migration |
| kustomization_path | prefix with `Eks/` | do NOT prefix — `Aks/` is already the default in the reusable workflow |

> **Single pattern**: EKS and AKS deploy **all environments via GitOps**. There is no more direct deploy. For namespace-level apps in ArgoCD (e.g.: `hotsite-b-prd` aggregates several components), use `resource_prefix: <component>` in `argocd-sync-deployment` to filter only the app Deployment.

### Migration note

- We are migrating apps from AKS to EKS
- On AKS, **all** environments (dev, hml, stg, prd) use GitOps
- On EKS, **all** environments (dev, hml, stg, prd) use GitOps
- The CI branch may be `feat/containerize` or `ci/<env>` during testing

## DNS — Automatic creation via GitHub Actions

### Workflow
- Repo: `my-org/k8s-infra`
- Workflow: `dns-update.yml`
- Trigger: `gh workflow run dns-update.yml --repo my-org/k8s-infra`

### Parameters
| Parameter | Values |
|-----------|--------|
| action | add, remove |
| environment | Desenvolvimento, Homologacao, Staging, Producao |
| cloud | "Azure (AKS)", "AWS (EKS)" |
| domains | comma-separated FQDNs |
| record_type | CNAME (default), A |
| dns_scope | Interno, Externo, Ambos |

### Rules
- **dev/hml**: dns_scope = `Interno`
- **stg/prd**: dns_scope = `Ambos`
- **Production**: ALWAYS ask the user before creating DNS
- Create DNS automatically when creating an HTTPRoute (except production)

### Default targets (CNAME)
| Environment | Cloud | Target |
|-------------|-------|--------|
| dev/hml | AKS | frontdoor-example.azurefd.net |
| stg/prd | AKS | frontdoor-example.azurefd.net |
| dev | EKS | internal-alb-example.us-east-1.elb.amazonaws.com |
| hml | EKS | cdn-example.cloudfront.net |
| stg/prd | EKS | cdn-example.cloudfront.net |

### Example
```bash
gh workflow run dns-update.yml --repo my-org/k8s-infra \
  -f action=add \
  -f environment=Homologacao \
  -f cloud="Azure (AKS)" \
  -f domains="myapp.hml.example.org" \
  -f record_type=CNAME \
  -f dns_scope=Interno
```
