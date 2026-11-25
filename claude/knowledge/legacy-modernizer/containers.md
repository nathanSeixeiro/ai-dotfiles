# Containers

## CONTAINER-CORE (.NET 5+)
Use the Dockerfile template and pipeline in app-onboarding.md (EKS or AKS model). Port: 8080 for .NET 8+, 80 for older. Autoscaling/PDB/ExternalSecret rules from app-onboarding apply. No version upgrade.

## CONTAINER-48 (.NET Framework 4.8, IIS in Windows container, AKS)

Template = `gateway` `docker/Dockerfile.iis` (generic via `ARG APP`). Copy it, do not reinvent. Key traits:
- Build INSIDE Docker (multi-stage): `dotnet/framework/sdk:4.8-windowsservercore-ltsc2022`, msbuild WebPublishMethod=Package to `C:\publish`. `/restore:false` works only if `packages/` is committed; otherwise add `nuget restore` (private feed token via secret mount/arg only for feed access, never app secrets).
- Runtime base picked by app type:
  - `dotnet/framework/aspnet:4.8-windowsservercore-ltsc2022`: WebForms / MVC / WebAPI
  - `dotnet/framework/wcf:4.8-windowsservercore-ltsc2022`: has `.svc` / WCF
  - `mcr.microsoft.com/windows/server:ltsc2022`: ONLY with a written reason (COM/CAPICOM, full Windows API). Needs manual IIS feature install; tramitacao-api is the example. Copy `ServiceMonitor.exe` from the aspnet image.
- 64-bit AppPool by default. If the app has x86 interop/COM: enable 32-bit (`appcmd set apppool DefaultAppPool /enable32BitAppOnWin64:true`) and say so.
- AppPool: idle timeout 0, periodic restart 0, `startMode AlwaysRunning` (pod controls lifecycle).
- Splunk OTel .NET for IIS: install with SHA256 check against `checksums.txt` (`Install-OpenTelemetryCore -LocalPath ... -SkipReleaseVerification`, `Register-OpenTelemetryForIIS`). Version `v1.16.0` unless user says otherwise.
- LogMonitor `v2.2.1` + `docker/LogMonitorConfig.json`; ENTRYPOINT `LogMonitor.exe ServiceMonitor.exe w3svc`. Deployment MUST NOT override ENTRYPOINT (ServiceMonitor passes env vars to `DefaultAppPool`; app must run in DefaultAppPool).
- Timezone `E. South America Standard Time`.
- Permissions: `IIS_IUSRS` read/execute on wwwroot; Modify only on `Uploads`/`App_Data` if they exist. Never Full on wwwroot (tramitacao does; do not copy).
- No secrets in the image. Runtime config:
  - `APPSETTING_<key>` -> appSettings (ConfigMap/Secret)
  - `CONNSTR_<name>` -> connectionStrings (Secret via ExternalSecret)
  - Web.config uses `Microsoft.Configuration.ConfigurationBuilders.Environment`. Copy the `configSections`/`configBuilders` blocks and the NuGet package reference from a gateway project's Web.config (source of truth; match package version). This is a CONFIG change in the app repo: show diff, get OK.
- `.dockerignore` required (`COPY . .` otherwise sends .git and local config): `.git`, `**/bin`, `**/obj`, `*.user`, `.vs`, `**/appsettings.*.local.json`.

### Pipeline CONTAINER-48 (AKS, GitOps)

Jobs: `docker` -> `deploy-gitops` -> `argocd-sync`. No separate msbuild job (build is in Dockerfile). Docker needs a Windows runner whose OS matches the base (`ltsc2022` -> `windows-2022`; do NOT use `windows-latest`, it can move to a newer Windows and break process isolation. Verify, or use a self-hosted Windows 2022 runner).

Environments: develop -> `dvp` (no Windows pool in dev: ask user), homolog -> `hml`, main -> `review` (input of update-image-tag, never `prd`). QA shares hml per app-onboarding.

```yaml
name: <App> - CI/CD

on:
  push:
    branches: [homolog, main]
    paths-ignore: ['**.md']
  workflow_dispatch:

permissions:
  contents: read

jobs:
  docker:
    name: Docker Build - [${{ github.ref_name }}]
    runs-on: windows-2022          # must match ltsc2022 base
    steps:
      - uses: actions/checkout@v4
      - name: Login to ACR
        env:
          REGISTRY_URL: ${{ github.ref_name == 'main' && secrets.REGISTRY_URL_PRD || secrets.REGISTRY_URL_HML }}
          REGISTRY_LOGIN: ${{ github.ref_name == 'main' && secrets.REGISTRY_LOGIN_PRD || secrets.REGISTRY_LOGIN_HML }}
          REGISTRY_PASSWORD: ${{ github.ref_name == 'main' && secrets.REGISTRY_PASSWORD_PRD || secrets.REGISTRY_PASSWORD_HML }}
        run: echo "$env:REGISTRY_PASSWORD" | docker login $env:REGISTRY_URL --username "$env:REGISTRY_LOGIN" --password-stdin
      - name: Build and push (no app secrets)
        env:
          REGISTRY_URL: ${{ github.ref_name == 'main' && secrets.REGISTRY_URL_PRD || secrets.REGISTRY_URL_HML }}
        run: |
          docker build --build-arg APP=<ProjectFolder> -t "$env:REGISTRY_URL/<image>:${{ github.run_number }}" -f docker/Dockerfile.iis .
          docker push "$env:REGISTRY_URL/<image>:${{ github.run_number }}"

  deploy-gitops:
    if: ${{ always() && needs.docker.result == 'success' }}
    needs: docker
    uses: my-org/devops-workflows/.github/workflows/update-image-tag.yml@main
    with:
      app: '<app>'
      environment: ${{ github.ref_name == 'homolog' && 'hml' || 'review' }}
      image: ${{ github.ref_name == 'homolog' && 'acrhml.azurecr.io/<image>' || 'acrprd.azurecr.io/<image>' }}
      tag: ${{ github.run_number }}
      kustomization_path: ${{ github.ref_name == 'homolog' && 'kubernetes/overlays/hml/ns-<app>-hml/<component>' || 'kubernetes/overlays/prd/ns-<app>-prd/<component>' }}
    secrets: inherit

  argocd-sync:
    if: ${{ !cancelled() && needs.deploy-gitops.result == 'success' }}
    needs: deploy-gitops
    uses: my-org/devops-workflows/.github/workflows/argocd-sync-deployment.yml@main
    with:
      argocd_app_name: ${{ github.ref_name == 'homolog' && '<app>-hml' || '<app>-prd' }}
      resource_prefix: <component>
    secrets:
      ARGOCD_SERVER: ${{ github.ref_name == 'homolog' && secrets.ARGOCD_SERVER_AKS_HML || secrets.ARGOCD_SERVER_AKS_PRD }}
      ARGOCD_USERNAME: ${{ github.ref_name == 'homolog' && secrets.ARGOCD_USERNAME_AKS_HML || secrets.ARGOCD_USERNAME_AKS_PRD }}
      ARGOCD_PASSWORD: ${{ github.ref_name == 'homolog' && secrets.ARGOCD_PASSWORD_AKS_HML || secrets.ARGOCD_PASSWORD_AKS_PRD }}
```
(Note: the docker job has NO `environment:` because it holds no app secrets. Registry secrets are repo-level.)

### GitOps side (AKS path `kubernetes/...`, no prefix)
Per app-onboarding checklist, plus for Windows: `nodeSelector: kubernetes.io/os: windows` (and tolerations if the pool is tainted; read an existing Windows deployment such as tramitacao in the gitops repo first), probes TCP/HTTP on 80, do not override ENTRYPOINT, ExternalSecret `data[].secretKey` = `CONNSTR_<name>` / `APPSETTING_<key>`, ConfigMap for plain `APPSETTING_*`. PDB/ScaledObject rules from app-onboarding. Windows pods are slow to start: set generous probe `initialDelaySeconds`.

### OTel
Splunk OTel for .NET on IIS is installed in the image (above). Apps with `dnsPolicy: None` need OTLP endpoint env via hostIP like app-onboarding, adapted for Windows; copy env block from the tramitacao deployment in gitops.

## Exception example: tramitacao-api
Uses `windows/server` base, 32-bit AppPool (`enable32BitAppOnWin64`), CAPICOM registered at build, build-arg secrets baked into Web.config (DO NOT COPY: leak), Full permission on wwwroot (DO NOT COPY), docker job with `environment: production` (unneeded once secrets are runtime).
