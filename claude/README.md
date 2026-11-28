# Claude Code personal config — architecture

Overview of how `CLAUDE.md`, `agents/`, and `skills/` fit together in this config.

## Layers

1. **`CLAUDE.md`** — always-loaded summary: communication style, Kubernetes/GitHub/Terraform defaults, production-safety rules, secrets handling, and the entry point into the agent system (`## Subagents`).
2. **`agents/`** — named subagent definitions, spawned via the `Task`/`Agent` tool for orchestrated multi-step work.
3. **`skills/`** — on-demand instruction sets, loaded by description match or invoked explicitly (`Skill` tool). Apply to the main session and to any agent.

Routing rule: for a single small/direct ask, just do it inline — no agent needed. Otherwise:
- dev/code request → `dev-lead`
- ops/infra request → `devops-lead`
- vague ask or 3+ tasks → `project-manager` first for a spec, then route per its plan

Leads orchestrate; they never do the specialist work themselves. Don't spawn a specialist directly — go through its lead, except when using a specialist standalone for a quick one-off (documented per-agent below).

## Agent roster

### Dev team

| Agent | Role | Spawned by | Can write code? | Can merge/push main? |
|---|---|---|---|---|
| `dev-lead` | Tech lead — breaks features into tasks, spawns dev/qa/review/validate/project-manager, runs the gate, delivers | user | no (orchestrates only) | no |
| `dev-agent` | Implements code + unit tests on a feature branch | dev-lead | yes | no |
| `idp-dev` | Senior dev for internal-platform/portal code (plugins, backend modules, scaffolder-style actions, infra-automation functions). Thinks before typing — contracts are expensive to get wrong | dev-lead | yes | no |
| `qa-agent` | Runs the repo's own test runner, reports pass/fail + coverage gaps | dev-lead | no (tests only) | no |
| `rules-validator` | Read-only audit of a branch/PR against hard rules (build, trailer, scope, secrets) — PASS/FAIL verdict | dev-lead | no | no |
| `project-manager` | Generic spec-writer/requirements analyst — turns requests into specs with acceptance criteria and task breakdowns; tracks status | dev-lead, devops-lead, or user directly | no | no |

Gate loop (`dev-lead`): dev-agent implements → qa-agent + `code-review` skill run in parallel on the diff → rules-validator audits → all green → PR opened (never merged) → delivered to user. Acceptance = rules-validator's verdict, not a subagent's claim.

### DevOps team

| Agent | Role | Spawned by | Can write code? | Can touch prod? |
|---|---|---|---|---|
| `devops-lead` | Tech lead — routes ops requests to the right specialist, runs the gate, delivers | user | no (orchestrates only) | no |
| `devops-agent` | General ops: CI/CD, pipelines, Dockerfiles, GitOps manifests, GitHub ops, debug/logs | devops-lead | yes (config/IaC) | no |
| `terraform-specialist` | Terraform: modules, import/moved, validate. Plan review comes from CI, never local apply | devops-lead | yes (`.tf` only) | no — read-only, hands PR to user |
| `k8s-troubleshooter` | Investigates cluster incidents: evidence-first RCA, fix via PR, never `kubectl edit/patch` on GitOps-managed resources | devops-lead | no (diagnosis + PR) | no — read-only |
| `platform-engineer` | Designs the internal developer platform (portal + infra-automation + GitOps): golden paths, templates, schemas. Hands code work to `idp-dev` via dev-lead | devops-lead | no (YAML/design only) | no |

Routing (`devops-lead`): vague/large request → project-manager first; `.tf` work → terraform-specialist; CI/CD/GitOps/Dockerfiles → devops-agent; cluster incident → k8s-troubleshooter; self-service/golden-path design → platform-engineer; anything needing application code → hand off to dev-lead.

### Cross-team

- A dev issue found during ops work becomes a task for `dev-lead`.
- Ops work needed by the dev team (e.g. a new pipeline) is built by the devops team.
- `idp-dev` and `platform-engineer` split by design-vs-code: platform-engineer designs (YAML/schema), idp-dev implements (code), both flow through their respective leads.

## Hard rules shared across all agents

- Never merge, never push to main/master, never force-push. Feature branches only (`feat/*`, `fix/*`), branched from up-to-date main.
- Never change `git user.name`/`user.email`, never `--author`. Commit trailer per the session's attribution instructions.
- GitHub writes via `gh` CLI. Always verify real PR/run state (`gh pr view`, `gh run view`) before claiming anything about it — never trust a cached assumption.
- Terraform locally: `init -backend=false` + `validate` only. Apply/destroy/plan is CI-only, or explicit human-run with confirmation.
- Production (or anything that looks like it): read-only by default. Any prod/DNS/destructive op needs explicit user sign-off — hand them the exact command.
- Secrets: never print/echo/log/commit values, never read `.env`/PEM/credential files unless explicitly told to. List names/keys only.
- Style: `caveman-mode` (short, direct, code over prose). Every task ends with 1-2 lines: what was skipped, what was not checked, risk.

## Skills

| Skill | Use |
|---|---|
| `caveman-mode` | Lazy-ops scope-discipline/blast-radius checklist for infra/K8s/Terraform/GitOps changes. Loads for ops work, not general chat. |
| `todo-discipline` | Tracking multi-step work with `TodoWrite` — item = outcome + owner + proof; gates are items too. Mandatory for leads on 3+ step or 2+ subagent work. |
| `workflow-path-triggers` | Scoping CI/CD `on.push.paths` in multi-app repos so one app's change doesn't trigger another's pipeline. |
| `spec-workflow` | Conventions for multi-step spec work: when to write one, where it lives, naming. |
| `security-audit` | Security review/vulnerability guidance for codebases, APIs, CLIs, libraries, daemons. |
| `enterprise-report` | Generates a print-ready corporate-style HTML/PDF report (status, RCA, analysis). |

Skills are available to the main session and every agent; agents reference the specific ones relevant to their role in their own frontmatter/body.

## Notes

- All agent/repo/cluster names here are generic placeholders — fill in real repo paths, cluster contexts, and stack details per project as you encounter them; nothing is hardcoded to a specific org anymore.
- Output files (specs, reports, scratch): `~/work/specs/`, `~/work/reports/`, ad hoc temp dirs cleaned up after the task.
- Not yet audited: `settings.json` permission entries may still reference MCP servers (e.g. Splunk, EKS/AKS contexts, Atlassian) tied to the deleted/retired tooling — check if you hit a permission error naming a tool that no longer applies.
