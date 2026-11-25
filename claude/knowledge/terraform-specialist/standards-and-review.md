# Terraform standards and review checklist

## Code standards (from repo steering/instructions)

- Azure: Azure Verified Modules first (`Azure/avm-res-<provider>-<resource>/azurerm`). AWS: official registry modules or the repo's own `modules/`.
- `terraform fmt` clean. Pin provider and module versions. No open ranges.
- Every variable has `type` + `description`. Every output has `description`. Secrets: `sensitive = true`, values come from Key Vault / Secrets Manager, never in code or tfvars.
- `for_each` over `count`. Computed values in `locals`.
- Env differences via `*.tfvars`, never branch-per-env. Logic stays the same across envs.
- Tags on everything: `project`, `environment`, `managed_by=terraform`.
- Identity: Azure Workload Identity / Managed Identity. AWS EKS Pod Identity (preferred over IRSA).
- Stateful resources (KV, storage, DB): `lifecycle { prevent_destroy = true }` unless the task is the destroy.
- Refactor without destroy: `moved {}` blocks. Adopt existing: `import {}` blocks (plan shows it). Never `terraform state rm/mv` by hand locally.
- Key Vault names: `kv-<d|h|q|p>-<app>`, max 24 chars. Terraform creates vault, RBAC (ESO reader/admin) and placeholder secrets. Real values are set outside Terraform.
- Security context, resource limits: for K8s manifests, not here (gitops repo).

## Local run policy (hard)

```
terraform init -backend=false
terraform fmt -recursive
terraform validate
```
That is all. No `plan` (needs backend + creds), no `apply`, no `destroy`, no `import`, no `state`, no `workspace`, no `force-unlock`. Plan output comes from CI: `gh pr checks <n>`, `gh run view <id> --log`, or the plan comment on the PR. On Windows/Crew: run via `wsl bash -lc "..."`.

Optional linters if installed: `tflint`, `trivy config .`, `terraform-docs`. Absent = say so, do not install.

## Plan review checklist (read the CI plan before approving anything)

1. Header: `Plan: X to add, Y to change, Z to destroy`. Any `destroy` or `must be replaced` (`-/+`): STOP, name the resource, explain why (ForceNew attribute), propose alternative (`moved`, in-place attribute, `create_before_destroy`).
2. Wrong workspace/state: state key in `init` output matches env (`dev-keyvaults.tfstate` etc.).
3. Unexpected diffs on untouched resources = drift or provider upgrade. Say which.
4. Sensitive values not printed in the comment.
5. Blast radius note: which envs (matrix), which downstream projects (apply order), which apps lose access if it fails.
6. prd: apply is human-only. Hand the PR + environment approval to the user.

## PR rules (infra repos)

- Branch from up-to-date main: `feat/<topic>` or `fix/<topic>`.
- Labels: cluster (`eks-dev|eks-hml|eks-prod|aks-dev|aks-hml|aks-prd`) + type (`security|migration|ci`) + resource (`terraform`, `frontdoor`, `keyvaults`).
- Commit trailer per work-rules. Never push to main.
- Check `gh pr view` for real state before commenting.
- One concern per PR. Pipeline path trigger updated if you add a project dir.

## Change recipes

**New Key Vault (app onboarding)**: file `Aks/terraform/keyvaults/environments/<env>/kv-<app>.tf` (QA -> inside `hml/`, name `kv-<app>-qa.tf`, vault `kv-q-<app>`). Copy the nearest sibling, change names only. Companion in gitops: ClusterSecretStore + ExternalSecret (not yours: tell devops-agent).

**New AWS app resources**: `accounts/<env>/<app>.tf` using `modules/{s3,iam,iam-policies,elastic-cache,security-group}`. Same file in dev -> hml -> prd, each its own PR.

**New GitHub environment**: `projects/<repo>/main.tf` from `projects/_template` using `modules/github_environment`. Secret values never in the file: use placeholders and set values outside Terraform.

**Import existing resource**: `import {}` block + matching resource, validate, let CI plan show "will be imported" with zero changes. Anything else in the plan = stop.

**Version bump (provider/module/EKS)**: one PR, read the changelog, validate, call out ForceNew risks. EKS upgrade order: control plane, addons, node groups. Spec examples in `k8s-infra/.kiro/specs/eks-dev-version-upgrade`.
