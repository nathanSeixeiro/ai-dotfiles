# Crossplane and Backstage patterns

## Crossplane (v2, as in the PoC)

- XRs are **namespaced**. Claims are not used the v1 way: the "claim" in the PoC is a namespaced XR yaml under `claims/<name>.yaml`, applied by ArgoCD.
- Compositions use **pipeline mode** with `function-go-templating` + `function-auto-ready`. No legacy `resources:` patch-and-transform.
- Provider: `provider-upjet-github` v0.20.0, namespaced managed resources (`*.github.m.upbound.io`). ProviderConfig credentials come from a Secret created by ExternalSecret / script, never committed.
- Ordering inside a Composition: dependent resources gated on readiness of earlier ones (ruleset only after files exist, otherwise the ruleset blocks the first commit).
- `deletionPolicy`/management policy: default to **not deleting** real-world resources (no `Delete`). Deleting a claim must not delete a GitHub repo or a cloud resource unless the user asked.
- Test without a cluster: `scripts/render.sh` (crossplane render with local binaries + fixtures `test/render/{xr,observed,functions}.yaml`). Run `render` for first reconcile AND with `observed` (resources Ready) before opening a PR.
- Validate XRD change: additive only (new optional field). Breaking field change = new XRD version, user sign-off.

### New Composition checklist

1. XRD: `spec.names`, scope `Namespaced`, `openAPIV3Schema` with required fields, enums, regex for names, defaults. Validate input at the XRD (trust boundary).
2. Composition pipeline: templates produce deterministic names; every resource labelled with the XR name.
3. Tags/labels: owner team, `managed_by=crossplane`, env.
4. Fixture + `render` output reviewed (diff of produced resources in the PR description).
5. Provider/function versions pinned in `00-packages.yaml`.
6. Permission: the provider credential has the minimum scope. Write down what it can do.
7. Rollback path written: how to detach (orphan) and hand the resource back.

## Backstage

### Software Template conventions

- `apiVersion: scaffolder.backstage.io/v1beta3`, `kind: Template`, tags, `spec.owner`.
- Parameters validate input: regex for names, enums for env, `RepoUrlPicker` with `allowedOrganizations: [my-org]`, `OwnerPicker` with `catalogFilter: {kind: Group}`.
- Steps produce a **PR** (never push to a protected branch). Output links to the PR and to the catalog entity.
- No secrets in template parameters or skeleton. Credentials come from the integration, not from the form.
- Skeleton files are templated (`${{ values.* }}`); keep skeletons minimal.
- Template changes: test with `templates/test-template` pattern or a dry run in the dev portal before registering.

### catalog-info.yaml conventions

- `metadata.name` = repo name (kebab-case). `spec.owner` = a real GitHub team (Group imported from the org, e.g. `devops-team`, `app-d`). `spec.system` = system name used in the catalog. `lifecycle`: `experimental|production|deprecated`.
- Annotations to add when the plugin exists: `github.com/project-slug: my-org/<repo>`, `backstage.io/techdocs-ref: dir:.`, Kubernetes label selector, ArgoCD app name.
- `docs/PROJECTS.md` (gitops) is the project -> cluster -> namespace source to cross-check ownership.

### Changing the portal

- Config-only change (app-config, catalog, template yaml): platform-engineer can do it via PR.
- New/changed **plugin, backend module, custom scaffolder action, frontend code**: that is code. Write the design (what, where, acceptance) and hand to `dev-lead` -> `idp-dev`.
- Upgrades: `yarn backstage-cli versions:bump`, one PR, read release notes, run `yarn tsc` + `yarn test`; new frontend system vs legacy app differs between PoC (new) and running portal: check `packages/app` before assuming.
- Prefer an existing community/`@backstage` plugin over custom code. Pin versions (Yarn lockfile committed).

## ArgoCD wiring

- `idp-claims` Application watches `claims/`. Merge to main = reconcile. CODEOWNERS (`devops-team`) is the approval control, so keep CODEOWNERS and the repo ruleset intact.
- Real clusters follow the gitops layout (`{Aks|Eks}/kubernetes/argocd/<env>/{projects,applications}`): new AppProject + Application, register both in the kustomization, update AppProject whitelist for any new kind (Crossplane XR kinds included).

## Working on the PoC locally

- `scripts/up.sh` (kind + ArgoCD + secrets), `backstage/bootstrap.sh` (once), `backstage/start.sh` (http://localhost:3000). Needs Docker + ~5 GB RAM, WSL with `kind`, `kubectl`, `jq`, Node 22/24.
- Context `kind-idp-poc` is a throwaway sandbox: kubectl reads fine. Teardown (`scripts/down.sh`) and anything touching GitHub org resources: ask the user.
- `.env` and PEM keys stay out of git. Never read them.
