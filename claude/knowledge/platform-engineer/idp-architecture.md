# IDP architecture (Backstage + Crossplane + ArgoCD)

Facts read from the repos 2026-11. Two things exist today: a **running Backstage** (dev) and an **IDP PoC** (local kind). They are not the same system.

## Repos

| Repo | Local path | What |
|---|---|---|
| `backstage-infra` | `~/work/devops/backstage-infra` | Backstage portal source (custom image). Yarn 4 monorepo, `packages/{app,backend}`, `templates/`, `catalog/`, Playwright config |
| `idp-poc` | `~/work/project/idp-poc` | PoC: Backstage form -> PR -> ArgoCD -> Crossplane -> GitHub repo |
| `k8s-infra` | `~/work/devops/terraform/k8s-infra` | Terraform `Aks/terraform/kubernetes/modules/backstage` + `Aks/backstage/{templates,catalog}` |
| `k8s-gitops` | `~/work/devops/k8s-gitops` | Where the apps (incl. Backstage) are deployed via ArgoCD overlays |

## Backstage (running, dev)

- URL `https://backstage.dev.example.org`. Custom image built by `.github/workflows/docker-build.yml` -> ACR. Reason: the official image failed with `NotImplementedError: No implementation available for apiRef{plugin.notifications.service}`.
- Config: `app-config.yaml` (local) + `app-config.production.yaml` (K8s overrides, local catalog paths in the image).
- Secrets by ExternalSecret from Key Vault `kv-dev-backstage-f957fa`: `GITHUB_APP_ID`, `GITHUB_APP_PRIVATE_KEY`, `GITHUB_APP_WEBHOOK_SECRET`. Never print values.
- Backend plugins: app, proxy, scaffolder (+github, +notifications modules), techdocs, auth (guest provider), catalog (+scaffolder-entity-model, +logs), permission (allow-all policy), search (pg, catalog, techdocs), kubernetes, notifications, signals, mcp-actions.
- Templates today: `create-isolado-sofia`, `delete-isolado-sofia`, `list-isolado-sofia` (dispatch workflows in `app-b`), `onboard-repo-catalog`, `register-app-catalog` (generate `catalog-info.yaml` + PR on the target repo), `test-template`.
- Catalog: 28 components in `catalog/catalog-info.yaml`, grouped by system (Sofia, SEED, EA, EB, HANDS, ...).

### Known limits of the running portal (raise them, do not silently fix)

- Auth = guest provider with `dangerouslyAllowOutsideDevelopment: true`. Not acceptable beyond dev. PoC already does GitHub login.
- DB = in-memory SQLite in `app-config.yaml` (`:memory:`): catalog is rebuilt on restart. Search module uses `pg`: confirm what prod really uses.
- Permission policy = allow-all.
- Only dev URL exists. No hml/prd Backstage yet.

## IDP PoC (local kind)

```
Backstage template --PR (user token)--> claims/<repo>.yaml
   merge (CODEOWNERS devops-team) --> ArgoCD (idp-claims) applies on kind
   --> Crossplane Composition `githubrepository` creates:
       Repository (topic idp-backstage), TeamRepository (owner maintain, devops-team admin),
       RepositoryFile catalog-info.yaml + .github/CODEOWNERS, RepositoryRuleset (after files)
   --> Backstage discovers catalog-info.yaml (<= 5 min)
```

Versions: Backstage 1.56.2 (new frontend system), Crossplane 2.4.3 (namespaced XR, pipeline Compositions), provider-upjet-github v0.20.0, function-go-templating v0.13.2, function-auto-ready v0.7.3, ArgoCD v3.5.6.

Layout: `platform/crossplane/{00-packages,10-xrd,20-composition,30-providerconfig}.yaml`, `platform/argocd/applications.yaml`, `backstage/{templates,overlay,app-config.idp.yaml,bootstrap.sh,start.sh}`, `claims/`, `kind/cluster.yaml`, `scripts/{up,down,render}.sh`, `test/render/*`.

Credentials (3 on purpose, least privilege): `idp-crossplane` (GitHub App, repo admin write, only Crossplane), `idp-reader` (GitHub App, read, catalog), `idp-login` (OAuth App, user login; PR is authored by the user's own token).

PoC limits: name prefix `poc-` mandatory, SQLite in memory, allow-all permission, ruleset without bypass actor, Composition has no `Delete` policy on purpose (repos are archived by hand), kind cluster only (`kind-idp-poc`).

## Gap PoC -> product (roadmap candidates, NOT approved)

Treat as options to present to the user with trade-offs. Smallest viable first.

1. Real auth (GitHub OAuth, as PoC) + real DB + permission policy on the running Backstage.
2. Promote the GitHub-repo golden path from kind to a real cluster (which cluster? needs user decision; Crossplane controller + provider creds via ExternalSecret).
3. Golden path "new app onboarding": today it is a manual checklist (`app-onboarding.md`: Key Vault in infra, overlay + ArgoCD project/app + ClusterSecretStore in gitops, pipeline). Candidate automation = Backstage template that opens PRs in the existing repos. Prefer PR generation over Crossplane for things already owned by Terraform/GitOps (no second source of truth).
4. Catalog completeness: only 28 components; `docs/PROJECTS.md` in gitops already lists project -> cluster -> namespace and could feed the catalog.
5. Observability links in the catalog (Splunk APM service page annotations) once the observability-specialist defines the naming.

## Boundaries

- Terraform owns: Key Vaults, clusters, Front Door, AWS resources. Crossplane must not take over those without an explicit user decision (state ownership conflict).
- GitOps owns: K8s manifests. Backstage/Crossplane produce PRs or claims, never `kubectl apply` on shared clusters.
- Prod: no Crossplane/Backstage change goes to a prd cluster without user sign-off.
