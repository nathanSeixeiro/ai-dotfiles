# IIS, Web App, and Azure DevOps migration

## Reading the source pipelines

- YAML pipelines (`azure-pipelines.yml`): read from the app repo.
- Classic build/release: user exports JSON from Azure DevOps UI (Export) into `~/work/__kiro-cli/temp/<app>/`. Variable groups: user lists names + non-secret values in the same folder. Secret values are NOT exportable (see triage-and-secrets.md).
- Extract: triggers/branches, stages -> environments, approvals, tasks (build, test, publish, deploy), agent pools -> runner labels, service connections, variable groups, schedules.

Mapping:

| Azure DevOps | GitHub Actions |
|---|---|
| Build pipeline / stages | jobs |
| Release stage + approval | job with `environment` (approval = `review` on main) |
| Agent pool | `runs-on` self-hosted label |
| Variable group (secret) | Environment secret (IIS) / Key Vault (k8s, Web App) |
| Variable group (plain) | Environment var (`vars.*`) / ConfigMap / App Setting |
| Service connection | repo/org secret (or OIDC) |
| Artifact | upload-artifact / download-artifact |

Keep the classic pipeline until the new one deploys successfully in hml; disable (do not delete) afterwards, and tell the user to do it.

## IIS (self-hosted runners, reference: app-a)

Facts from sisae: runner labels `BuildP1` (build), `SISAE-H`, `SISAE-P1`, `SISAE-P2` (deploy). VS2017 MSBuild located with vswhere, NuGet 4.1.0, app pool control via `C:\Scripts\ManageAppPools.ps1 -appPoolName <name> -action stop|start`, `web.config` edited by XPath from GH Environment secrets/vars, destination wiped then copied.

Fix these when migrating (sisae problems): publish with `Release` (sisae uses debug on main); no copy-pasted P1/P2 jobs (use one deploy job with a runner matrix); no silent skip of missing secrets; add backup, health check, rollback; no token in git clone URL (use `actions/checkout` if the runner allows it, else keep and note).

Runners not yet installed: list the labels used and flag them; do not assume they exist.

Environment: homolog -> `homolog`; main approval+prd secrets -> `review` (decision option 1). Job-level `environment: review` on the prd deploy job. No separate approval job.

Template (adapt names, paths, keys):

```yaml
name: <App> - CI/CD

on:
  push:
    branches: [homolog, main]
  workflow_dispatch:

permissions:
  contents: read

env:
  IIS_APP_PATH: ${{ vars.APPPATH }}

jobs:
  build:
    name: Build - [${{ github.ref_name }}]
    runs-on: BuildP1
    steps:
      - uses: actions/checkout@v4
      - uses: nuget/setup-nuget@v2
      - name: Restore
        shell: powershell
        run: nuget restore <Solution>.sln
      - name: Build and publish
        shell: powershell
        run: |
          # locate MSBuild (vswhere) as in sisae; fail if missing
          & $msbuild <Project>/website.publishproj /p:DeployOnBuild=true /p:PublishProfile="<profile>" /p:Configuration=Release /p:Platform="AnyCPU"
      - uses: actions/upload-artifact@v4
        with:
          name: <app>-package
          path: <publish output>
          retention-days: 30

  deploy:
    name: Deploy - ${{ matrix.runner }}
    needs: build
    environment: ${{ github.ref_name == 'main' && 'review' || 'homolog' }}   # review = approval gate + prd secrets
    runs-on: ${{ matrix.runner }}
    strategy:
      max-parallel: 1              # one server at a time
      fail-fast: true
      matrix:
        runner: ${{ fromJSON(github.ref_name == 'main' && '["<PRD-1>","<PRD-2>"]' || '["<HML>"]') }}
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: <app>-package
          path: ${{ github.workspace }}/artifacts

      - name: Stop app pool
        shell: powershell
        run: '& C:\Scripts\ManageAppPools.ps1 -appPoolName "${{ vars.APPPOOLNAME }}" -action stop'

      - name: Backup current site
        shell: powershell
        run: |
          $bk = "C:\deploy-backup\<app>\${{ github.run_number }}"
          New-Item -ItemType Directory -Path $bk -Force | Out-Null
          Copy-Item -Path "$env:IIS_APP_PATH\*" -Destination $bk -Recurse -Force

      - name: Replace Web.config values (fails on missing)
        shell: powershell
        env:                                  # step-level: secrets only here
          AmbienteDeDados: ${{ vars.AMBIENTEDEDADOS }}
          ConnectionStringBase: ${{ secrets.CONNECTIONSTRINGBASE }}
          # one line per key from the inventory map
        run: |
          $ErrorActionPreference = 'Stop'
          $path = "${{ github.workspace }}\artifacts\web.config"
          if (-not (Test-Path $path)) { throw "web.config not found" }
          $required = @{
            appSettings       = @{ AmbienteDeDados = $env:AmbienteDeDados; ConnectionStringBase = $env:ConnectionStringBase }
            connectionStrings = @{ }
          }
          foreach ($section in $required.Keys) { foreach ($k in $required[$section].Keys) {
            $v = $required[$section][$k]
            if ([string]::IsNullOrEmpty($v)) { throw "Missing required value: $section/$k" }
            Write-Output "::add-mask::$v"
          } }
          $xml = New-Object System.Xml.XmlDocument
          $xml.PreserveWhitespace = $true
          $xml.Load($path)
          foreach ($k in $required.appSettings.Keys) {
            $n = $xml.SelectSingleNode("//appSettings/add[@key='$k']")
            if (-not $n) { throw "appSettings key not in web.config: $k" }
            $n.SetAttribute('value', $required.appSettings[$k])
          }
          foreach ($k in $required.connectionStrings.Keys) {
            $n = $xml.SelectSingleNode("//connectionStrings/add[@name='$k']")
            if (-not $n) { throw "connectionString not in web.config: $k" }
            $n.SetAttribute('connectionString', $required.connectionStrings[$k])
          }
          $xml.Save($path)
          Write-Output "web.config updated (values not shown)"

      - name: Deploy files
        shell: powershell
        run: |
          $ErrorActionPreference = 'Stop'
          if (-not (Test-Path $env:IIS_APP_PATH)) { throw "IIS path not found: $env:IIS_APP_PATH" }
          Get-ChildItem $env:IIS_APP_PATH -Recurse -Force | Remove-Item -Recurse -Force
          Copy-Item "${{ github.workspace }}\artifacts\*" $env:IIS_APP_PATH -Recurse -Force

      - name: Start app pool
        shell: powershell
        run: '& C:\Scripts\ManageAppPools.ps1 -appPoolName "${{ vars.APPPOOLNAME }}" -action start'

      - name: Health check
        id: health
        shell: powershell
        run: |
          $url = "${{ vars.HEALTH_URL }}"
          for ($i = 1; $i -le 10; $i++) {
            try { $r = Invoke-WebRequest -Uri $url -UseBasicParsing -TimeoutSec 15; if ($r.StatusCode -lt 500) { Write-Output "OK $($r.StatusCode)"; exit 0 } } catch {}
            Start-Sleep -Seconds 10
          }
          throw "Health check failed: $url"

      - name: Rollback on failure
        if: ${{ failure() }}
        shell: powershell
        run: |
          $bk = "C:\deploy-backup\<app>\${{ github.run_number }}"
          if (Test-Path $bk) {
            & C:\Scripts\ManageAppPools.ps1 -appPoolName "${{ vars.APPPOOLNAME }}" -action stop
            Get-ChildItem $env:IIS_APP_PATH -Recurse -Force | Remove-Item -Recurse -Force
            Copy-Item "$bk\*" $env:IIS_APP_PATH -Recurse -Force
            & C:\Scripts\ManageAppPools.ps1 -appPoolName "${{ vars.APPPOOLNAME }}" -action start
            Write-Output "Rolled back to run ${{ github.run_number }} backup"
          }
```
Notes: `vars.APPPATH`, `APPPOOLNAME`, `HEALTH_URL` are per-environment vars. Keep `max-parallel: 1` so one server stays up. Prune old backups (keep last N) if disk is a concern: say so in PR, do not add silently.

Manual checklist (print for the user):
- Create GH Environment `review` with required reviewers; `homolog` without reviewers.
- Copy prd secrets/vars from `production` into `review` (names listed from inventory).
- Confirm runner labels exist and the service account can run `ManageAppPools.ps1`.
- Re-enter Library secrets that could not be exported.

## WEBAPP (reuse reusables)

Reuse `build-dotnet-framework.yml` (zip package, inputs `package_name`, `runs_on`, `solution_path`) and `deploy-azure-webapp.yml` (inputs `environment`, `app_name`, `slot_name`, `resource_group`, `package_name`, `artifact_name`, `runs_on`; secret `AZURE_CREDENTIALS`, optional `XML_SUBSTITUTION_VARS`).

Secrets via Key Vault reference (decision): do NOT use `XML_SUBSTITUTION_VARS` for secrets. Configure App Settings once (portal/az/terraform), not in the pipeline:
- Web App has system-assigned managed identity with `Key Vault Secrets User` on the KV (infra PR; plan/apply in CI only).
- App Setting value: `@Microsoft.KeyVault(SecretUri=https://<kv>.vault.azure.net/secrets/<Name>)` (no version).
- Connection strings go in the Connection strings section with the same name as in web.config; appSettings keys must match the web.config key.
- If KV has a firewall/private endpoint: Web App needs VNet integration. Slots: slot-sticky settings, give each slot identity access.
- Check in portal that references show as resolved.

Caller template:

```yaml
name: <App> - CI/CD

on:
  push:
    branches: [develop, homolog, main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  build:
    uses: my-org/devops-workflows/.github/workflows/build-dotnet-framework.yml@main
    with:
      solution_path: "<Solution>.sln"
      package_name: "<Site>.zip"
      runs_on: windows-latest        # or self-hosted label if build needs old VS
  deploy:
    needs: build
    uses: my-org/devops-workflows/.github/workflows/deploy-azure-webapp.yml@main
    with:
      environment: ${{ github.ref_name == 'main' && 'review' || github.ref_name == 'homolog' && 'homolog' || 'develop' }}
      app_name: <webapp-name>
      slot_name: ${{ github.ref_name == 'main' && 'Production' || github.ref_name }}
      resource_group: <rg>
      package_name: "<Site>.zip"
    secrets:
      AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}
```
(`environment` here is an INPUT of the reusable; no job-level `environment:` on the `uses:` job. `review` holds the approval gate and `AZURE_CREDENTIALS` for prd.)

Known gaps in `deploy-azure-webapp.yml` (read it before relying; propose a PR to devops-workflows, do not patch silently):
- `if: env.XML_VARS != ''` uses a step-level env var inside the same step's `if`; step env is not visible to its own `if`, so substitution is likely skipped. Verify with a test run.
- JSON-derived values not masked individually (`::add-mask::`).
- No health check, no slot swap, `azure/login` uses SP creds JSON (OIDC preferred).

## Manual checklist WEBAPP
- KV + RBAC infra PR merged; App Settings set to KV references.
- GH Environment `review` (reviewers) with `AZURE_CREDENTIALS`; dev/homolog environments.
- Re-enter Library secrets in KV.
