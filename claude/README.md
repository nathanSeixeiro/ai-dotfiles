# Claude Code configuration

Personal Claude Code setup. Hierarchical overview: models → instructions → agents → skills → settings.

## 1. Models

| Alias | Model ID | Used by |
|---|---|---|
| `opus` | `claude-opus-5-5` | `ANTHROPIC_DEFAULT_OPUS_MODEL` |
| `sonnet` | `claude-sonnet-5-5` | `ANTHROPIC_DEFAULT_SONNET_MODEL` |
| `haiku` | `claude-haiku-5-5` | `ANTHROPIC_DEFAULT_HAIKU_MODEL`, default session model (`settings.json` → `model`) |

- Default session model: `haiku`, effort `low` (`settings.json` → `modelSettings`).
- Agent models are set per agent in frontmatter (`model:` / `effort:`). Currently `qa-agent` and `rules-validator` run on `haiku` with `low` effort. All others inherit the session model.
- Model IDs must match exactly (`claude-sonnet-5-5`, not `claude-sonnet-5`) or `modelSettings` silently does nothing.

## 2. Instruction layers

1. `CLAUDE.md` — always loaded. Communication style, Kubernetes/GitHub/Terraform defaults, production-safety rules, secrets handling, subagent entry point.
2. `agents/` — named subagents, spawned for orchestrated multi-step work.
3. `skills/` — on-demand instruction sets, loaded by description match or invoked explicitly.

Rule of thumb: a single small or direct request is done inline. Multi-step work goes through a lead.

## 3. Agents

### 3.1 Routing

- Development request → `dev-lead`
- Ops/infra request → `devops-lead`
- Vague request or 3+ tasks → `project-manager` first (spec), then route per its plan

Leads orchestrate and never do specialist work themselves.

### 3.2 Development team

- `dev-lead` — breaks features into tasks, spawns workers, runs the gate, delivers. Does not write code.
  - `dev-agent` — implements code and unit tests on `feat/*` or `fix/*`.
  - `idp-dev` — internal-platform code (plugins, modules, actions, automation functions).
  - `qa-agent` — runs the repo's test runner, reports pass/fail. (haiku, low)
  - `rules-validator` — read-only PASS/FAIL audit against hard rules. (haiku, low)
  - `project-manager` — writes specs with acceptance criteria; tracks status.

Gate loop: dev-agent implements → qa-agent and the `code-review` skill run on the diff → rules-validator audits → PR opened. Never merged by an agent.

### 3.3 DevOps team

- `devops-lead` — routes ops requests, runs the gate, delivers. Does not do ops work.
  - `devops-agent` — CI/CD, pipelines, Dockerfiles, GitOps manifests, GitHub ops.
  - `terraform-specialist` — `.tf` work. Plans come from CI; local = `init -backend=false` + `validate` only.
  - `k8s-troubleshooter` — evidence-first incident RCA; fixes go through PRs, not `kubectl edit`.
  - `platform-engineer` — internal platform design (golden paths, templates, schemas). Code goes to `idp-dev` via dev-lead.

### 3.4 Cross-team

- Dev issues found in ops work → task for `dev-lead`.
- Ops work needing code (e.g. new pipeline features) → built by the devops team.
- `platform-engineer` designs, `idp-dev` implements.

## 4. Hard rules (all agents)

- No merges. No pushes to `main`/`master`. No force-push. Feature branches branched from up-to-date main.
- Git identity is never changed. No `--author`. Attribution follows the session's commit/PR instructions.
- GitHub writes via `gh`. Verify real PR/run state (`gh pr view`, `gh run view`) before stating anything about it.
- Terraform: `init -backend=false` + `validate` locally. Apply/destroy/plan only in CI, or by a human with confirmation.
- Production (or anything that looks like it): read-only. Prod, DNS, destructive, or merge actions need explicit user sign-off; the exact command is handed over.
- Secrets: never printed, logged, or committed. Never read `.env`, PEM, or credential files unless explicitly asked. List names only.
- Output style: short, direct. Every task ends with what was skipped, what was not checked, and the risk.

## 5. Skills

| Skill | Purpose |
|---|---|
| `caveman-mode` | Scope-discipline and blast-radius checklist for infra/K8s/Terraform/GitOps changes |
| `todo-discipline` | Multi-step tracking with `TodoWrite`; items are outcome + owner + proof |
| `workflow-path-triggers` | Scope CI `on.push.paths` in multi-app repos |
| `spec-workflow` | When and where to write specs for multi-step work |
| `security-audit` | Security review and vulnerability guidance |
| `enterprise-report` | Print-ready HTML/PDF report generator |

## 6. Settings

`settings.json` holds: model env vars (section 1), default model and effort, `modelSettings`, theme, MCP servers, and permissions.

Open item: `mcpServers` and permission entries may still reference tools from retired setups (Splunk, EKS/AKS, Atlassian). Audit before relying on them.

## 7. Files and output locations

- Specs: `~/work/specs/`
- Reports: `~/work/reports/`
- Scratch: removed by the agent that created it, once the task is done.
