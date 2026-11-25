# Triage + Secrets/Vars

## Triage decision tree

Read: csproj `TargetFramework` / `TargetFrameworkVersion`, packages.config, Web.config, *.sln, existing pipelines.

| Finding | Verdict |
|---|---|
| `net5.0`+ / `netcoreapp` / `net6+` | CONTAINER-CORE (Linux). Pattern: app-onboarding Dockerfile, port 8080 for .NET 8+, 80 for older |
| .NET Framework 4.8 (or 4.7.x only when user asks to move to 4.8) web app, containerizable | CONTAINER-48 (Windows container, AKS Windows pool `winhml`/prd). Not available for dev: no Windows pool in dev, ASK |
| .NET Framework with blockers (see below) or server not container-ready | IIS (self-hosted runner deploy) |
| Already on Azure Web App, or destination is Web App | WEBAPP |

Blockers for container (write the reason in the PR): Windows services / scheduled exe, machine-specific certs or GAC installs, local disk state outside Uploads/App_Data, MSMQ/DTC, non-HTTP listeners, license-bound components. COM/CAPICOM/32-bit interop is NOT a blocker but changes the base image (see containers.md).

If unsure: ask the user, do not guess.

Never upgrade: 4.7 -> 4.8 only if the user asks. Core 5/6/7 are EOL: flag, do not migrate versions.

## Secret vs var classification

- SECRET: passwords, connection strings, tokens, API keys, SMTP password, blob keys, machineKey, client secrets. When in doubt: secret.
- VAR (plain): environment name, URLs, hostnames, ports, feature flags, SMTP server/port, account display names.
- Rule: a value that appears in a connection string is secret as a whole.

## Matrix: where values live and how they are injected

| Target | Secrets | Plain vars | Injection |
|---|---|---|---|
| k8s (core + 4.8) | Azure Key Vault `kv-<d/h/p>-<app>` via ExternalSecret | ConfigMap | Runtime env vars. 4.8: `APPSETTING_<key>`, `CONNSTR_<name>` read by ConfigurationBuilders.Environment |
| Web App | Azure Key Vault + Key Vault reference in App Settings | App Settings | `@Microsoft.KeyVault(SecretUri=https://<kv>.vault.azure.net/secrets/<Name>)`. Web App managed identity has `Key Vault Secrets User`. No code change |
| IIS | GitHub Environment secrets | GitHub Environment vars | pwsh XML edit of Web.config at DEPLOY time on the target, never at build |

Naming: KV secret names use `--` (e.g. `ConnectionStrings--db`); k8s/env uses `__`. KV name max 24 chars.

## Guardrails: secrets and vars (MOST IMPORTANT)

1. Inventory first. Every key gets a row: name | secret/var | source (Library/pipeline/web.config) | destination (KV/GH env) | per-env values exist? (dev/hml/prd).
2. Missing value for an environment = STOP and list it. Never deploy with an empty/default value. Deploy scripts must FAIL when a required secret is empty (do not silently skip like sisae does today).
3. No secrets in: Dockerfile (ARG/ENV/RUN), docker build-args, committed web.config/appsettings, workflow YAML, artifacts, logs, PR text, chat output.
4. Mask: any secret derived at runtime (JSON fields, KV fetch) gets `::add-mask::` before use.
5. Least exposure: secrets only in the deploy step that needs them (step-level `env:`), not job-level.
6. Separate secrets and vars in GitHub: `secrets.*` vs `vars.*`.
7. Do not set or change secret values. Agent lists names; user sets values (`gh secret set`, KV portal/terraform).
8. KV for new envs needs TWO PRs: infra (`Aks/terraform/keyvaults/environments/<env>/kv-<app>.tf`) and, for k8s, gitops (ClusterSecretStore + ExternalSecret). For Web App, infra PR also adds RBAC for the Web App identity on the KV.
9. Rotation: k8s/Web App = change in KV (Web App: omit version in URI, up to ~24h lag, restart to force). IIS = change GH Environment secret then re-run deploy.

## Azure DevOps Library migration

- Library secret variables CANNOT be exported or read back. Only names and non-secret values are exportable. Secret values must be re-entered by the user from their original source.
- Procedure: list variable groups used by the pipelines -> table (name, secret?, per-stage scope) -> map to GH Environment (IIS) or KV (k8s/Web App) -> output the list of secrets the user must re-enter -> fail the migration validation until each exists.
- Variable group scoped per release stage = one value per GH environment (develop/homolog/production-or-review).
