# Requirements and backlog

Team context: small devops + dev team, one human owner (the user, not a programmer) who decides, merges and approves prod. Work is mostly infra/GitOps/CI/observability/platform, some .NET and TS/Go code.

## Team roster (who does what)

| Agent | Team | Takes |
|---|---|---|
| devops-lead | devops | routes ops work, gate, PR |
| devops-agent | devops | CI/CD, pipelines, Dockerfiles, GitOps manifests, GitHub ops |
| k8s-troubleshooter | devops | AKS/EKS incident RCA |
| legacy-modernizer | devops | legacy .NET to containers/GitHub Actions |
| terraform-specialist | devops | Terraform (k8s-infra, aws-infra, gh-env): code + validate, plan via CI |
| observability-specialist | devops | Splunk (logs/APM/detectors), OTel instrumentation |
| platform-engineer | devops | IDP design: Backstage config/templates, Crossplane XRD/Composition, golden paths |
| dev-lead | dev | orchestrates code work and the gate |
| dev-agent | dev | code + unit tests (.NET, JS/TS, Go, bash, PowerShell) |
| idp-dev | dev | Backstage plugins/backend modules, scaffolder actions, Crossplane functions |
| qa-agent / semantic_reviewer / rules-validator | dev | test, review, rules audit |
| project-manager | cross | requirements, task breakdown, PROJ cards, status |

Human-only: merge, Terraform apply, prod change, DNS, destructive ops, secrets values.

## Mode 1: Requirements (before work starts)

Trigger: new request, vague ask, or 3+ steps. Output: a spec file at `~/work/__kiro-cli/specs/<feature>.md` (kebab-case, no date).

Steps:
1. **Restate** the goal in 1-2 lines. Who benefits. Why now.
2. **Ask only what blocks.** Max 5 questions, each with a proposed default so the user can answer "ok". Do not ask what the repos/gitops/Jira can answer: read first.
3. **Scope**: in / out. Environments touched (dev/hml/qa/prd). Repos touched. Clusters (aks-*/eks-*).
4. **Acceptance criteria**: testable, one behavior each. Form: `WHEN <condition> THE SYSTEM SHALL <result>` or `GIVEN/WHEN/THEN`. Each criterion has a verification command or observable (a Splunk query, `terraform validate`, `gh pr checks`, an ArgoCD Healthy status).
5. **Risks and blast radius**: prd impact, shared state, data/secret exposure, rollback.
6. **Tasks**: ordered, each with owner agent, input, output, proof, dependencies, size (S <1h, M half-day, L day+; L must be split).
7. **Human gates** as explicit tasks: review, merge, apply, prod sign-off.
8. **Open questions** and assumptions, labeled.

Spec skeleton:
```markdown
# <Feature>
Status: draft | approved | in-progress | done
Card: PROJ-<n>

## Goal
## Scope (in / out)
## Environments and repos
## Acceptance criteria
- [ ] WHEN ... THE SYSTEM SHALL ... (verify: <command>)
## Risks / blast radius / rollback
## Tasks
| # | Task | Owner | Depends | Size | Proof |
## Human gates
## Open questions / assumptions
```

Definition of Ready: goal clear, scope + envs known, acceptance criteria testable, owner per task, no unanswered blocker.
Definition of Done: acceptance criteria verified with evidence, PR open with labels and trailer, gate verdicts attached, card updated, user informed. "Done" in Jira only when the user says so.

## Mode 2: Tracking (while work runs)

- Card = control point. Reuse an existing card if one matches (search first). New card goes to BACKLOG, started work to In Progress, Done only on explicit user word.
- Card rules are in steering `jira-cards.md`: project `PROJ`, type `Task`, title `[PROJECT] topic`, description 1-2 lines + `PR:`/`Issue:`/`Ref:`.
- On each stop point add a one-line comment: `Step: <what happened>. Blocked by: <what>. Owner: <user|CI|agent>. Next: <what>`. Check real PR/CI state first (`gh pr view`, `gh run view`).
- Touch only BACKLOG / In Progress / Done. Never SCHEDULED, FROZEN, OVERDUE, THIRD-PARTY PENDING.
- Confirm real status names on the board on first operation (e.g. "In Progress" vs a localized name) before transitioning.
- Weekly style summary on request: JQL by status, list stale In Progress (no comment for 7 days), list waiting-on-user items first.

## Splitting rules

- One PR = one concern = one cluster/env group when possible (gitops change-request rule: one change per PR).
- Infra order matters: Terraform kubernetes -> frontdoor/keyvaults; Key Vault before gitops ExternalSecret; app repo pipeline before overlay bump.
- Code request from ops side -> task owned by dev-lead, with acceptance criteria and repo/branch hints.
- Platform ideas (Backstage/Crossplane): split design (platform-engineer) from code (idp-dev). Design first, user approves, then code.

## Estimation hints (rough, say they are rough)

S: config value, label, doc, single-file manifest. M: new app onboarding (KV + overlay + pipeline), OTel rollout for one namespace, new Terraform file from sibling. L: new Terraform project, new Composition + tests, Backstage plugin, cluster upgrade. XL: split it.
