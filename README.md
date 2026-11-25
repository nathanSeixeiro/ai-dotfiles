# dotfiles-ai-config

Configs pessoais usadas com **Kiro CLI**, **GitHub Copilot CLI** e **Claude Code**.

## Estrutura

```
claude/
  CLAUDE.md                 # project memory sempre carregada (~/.claude/CLAUDE.md): resumo das regras always-on
  agents/                   # subagents (~/.claude/agents/*.md): port dos 13 agents Kiro (sem aws-specialist)
  skills/                   # skills (~/.claude/skills/*/SKILL.md): enterprise-report, task-handoff + port de todo o steering Kiro (work-rules, caveman-mode, jira-cards, spec-workflow, workflow-path-triggers, todo-discipline, dev-team, playbooks, app-onboarding). gmud-create NÃO portado (processo ITSM específico da empresa)
  knowledge/                # docs de referência por agent (~/.claude/knowledge/<agent>/*.md), igual ao kiro/knowledge sem aws-specialist
  settings.json             # hooks (block-secret-reads, guard-destructive-cmds) + mcpServers (~/.claude/settings.json)
copilot/
  copilot-instructions.md   # instructions globais do Copilot CLI (~/.copilot/copilot-instructions.md)
  agents/                   # custom agents (~/.copilot/agents/*.agent.md): devops-team + port dos agents Kiro (8)
  skills/                   # skills (~/.copilot/skills/*/SKILL.md): inclui steering do Kiro portado (work-rules, workflow-path-triggers, jira-cards, dev-team, playbooks)
kiro/
  agents/                   # agents (~/.kiro/agents/*.json): devops-lead + devops-agent, legacy-modernizer, k8s-troubleshooter, terraform-specialist, observability-specialist, platform-engineer; dev-lead + dev-agent, idp-dev, qa-agent, rules-validator; project-manager (cross-team)
  prompts/                  # system prompts dos agents (~/.kiro/prompts/*.md) + habblaud-install.md
  knowledge/                # docs de referência por agent (~/.kiro/knowledge/<agent>/*.md)
  steering/                 # steering docs (~/.kiro/steering/*.md)
  skills/                   # skills (~/.kiro/skills/<name>/SKILL.md): enterprise-report (HTML/PDF reports)
  settings/                 # mcp.json (token = placeholder, preencher local), cli.json (~/.kiro/settings/)
  skills/                   # skills (~/.kiro/skills/*/SKILL.md): enterprise-report, gmud-create, task-handoff
  hooks/                    # hooks (~/.kiro/hooks/*.json): guard-destructive-cmds, block-secret-reads
bin/
  kiro-cli-login            # login Kiro CLI via Windows no WSL (instalar em ~/bin)
```

## Agents (Kiro)

| Agent | Uso |
|---|---|
| `devops-agent` | EKS/AKS, Terraform, GitOps, GitHub. Read-first, write via CLI |
| `legacy-modernizer` | Migrar apps .NET legadas: containers (Core, 4.8 Windows), IIS, Web App, pipelines classic/Azure -> GitHub Actions. Guardrails de secrets/vars |
| `k8s-troubleshooter` | Investigar erros de apps em AKS/EKS: gitops, repo da app, infra, DB/DNS. Mesmas permissões do devops-agent + deny (mutações em prd, `az keyvault secret show`) |
| `dev-agent` | Implementa código + testes unitários em `feat/*`/`fix/*`. Sem merge, sem push em main |
| `qa-agent` | Roda o test runner do repo, reporta pass/fail + lacunas de cobertura. Não edita código |
| `rules-validator` | Auditoria read-only de branch/PR vs work-rules (build, labels, trailer, escopo, secrets). Veredito PASS/FAIL |
| `dev-lead` | Tech lead do time de dev. Usa todo list. Quebra feature em tarefas, spawna dev-agent / idp-dev / qa-agent / semantic_reviewer / rules-validator / project-manager, roda o gate, entrega. Não escreve código, nunca faz merge |
| `devops-lead` | Tech lead do time de devops. Usa todo list. Roteia o trabalho ao especialista certo (devops-agent / terraform-specialist / observability-specialist / platform-engineer / k8s-troubleshooter / legacy-modernizer / project-manager), roda o gate, entrega. Versão Windows/Crew em `kiro-crew/` |
| `terraform-specialist` | Terraform (`k8s-infra`, `aws-infra`, `terraform-gh-env`): AVM, import/moved, validate, leitura do plan do CI. Local = `init -backend=false` + `validate`; nunca plan/apply |
| `observability-specialist` | Splunk primeiro: SPL (índices `ix_{dvp,hml,prd}_{aks,eks}`), Splunk O11y APM/detectors, instrumentação OTel via PR no gitops. Read-only nos clusters |
| `platform-engineer` | IDP: Backstage + Crossplane + ArgoCD. Desenha golden paths, escreve template/XRD/Composition (YAML), testa com `render`; código vai pro `idp-dev` |
| `idp-dev` | Dev sênior do IDP (time dev): plugins/módulos Backstage (TS), scaffolder actions, Crossplane functions (Go). Planeja antes, reusa upstream, testa. Sem merge, sem push em main |
| `aws-specialist` | Consultor AWS que qualquer agent chama (subagent). Conhece `development-us`, `homologation-us`, `production-us`, `network-us`, TGW/VPC/endpoints/Resolver. Só leitura, com evidência; mudança vai pro dono (terraform-specialist / devops-agent) |
| `project-manager` | Requisitos + gestão de tarefas (um agente, dois modos): spec com critérios de aceite, quebra por agente, cards PROJ, status. Sem código, nunca marca Done |

Iniciar: `kiro-cli chat --agent <nome>` e descrever o problema numa mensagem normal.

## Agents (Claude Code)

Mesmos 13 agents do Kiro, sem `aws-specialist` (e sem as seções "AWS consultant" que cada agent tinha para chamá-lo — removidas no port). Orquestração via tool `Task` em vez da tool `subagent` do Kiro; `todo_list` virou `TodoWrite`; steering virou skills (`claude/skills/*`), carregadas por descrição em vez de `inclusion: always/manual/fileMatch`.

| Agent | Uso |
|---|---|
| `devops-agent` | EKS/AKS, Terraform, GitOps, GitHub. Read-first, write via CLI |
| `legacy-modernizer` | Migrar apps .NET legadas: containers (Core, 4.8 Windows), IIS, Web App, pipelines classic/Azure -> GitHub Actions. Guardrails de secrets/vars |
| `k8s-troubleshooter` | Investigar erros de apps em AKS/EKS: gitops, repo da app, infra, DB/DNS. Mesmas permissões do devops-agent + deny (mutações em prd, `az keyvault secret show`) |
| `dev-agent` | Implementa código + testes unitários em `feat/*`/`fix/*`. Sem merge, sem push em main |
| `qa-agent` | Roda o test runner do repo, reporta pass/fail + lacunas de cobertura. Não edita código |
| `rules-validator` | Auditoria read-only de branch/PR vs work-rules (build, labels, trailer, escopo, secrets). Veredito PASS/FAIL |
| `dev-lead` | Tech lead do time de dev. Usa TodoWrite. Quebra feature em tarefas, spawna dev-agent / idp-dev / qa-agent / semantic_reviewer / rules-validator / project-manager, roda o gate, entrega. Não escreve código, nunca faz merge |
| `devops-lead` | Tech lead do time de devops. Usa TodoWrite. Roteia o trabalho ao especialista certo (devops-agent / terraform-specialist / observability-specialist / platform-engineer / k8s-troubleshooter / legacy-modernizer / project-manager), roda o gate, entrega |
| `terraform-specialist` | Terraform (`k8s-infra`, `aws-infra`, `terraform-gh-env`): AVM, import/moved, validate, leitura do plan do CI. Local = `init -backend=false` + `validate`; nunca plan/apply |
| `observability-specialist` | Splunk primeiro: SPL (índices `ix_{dvp,hml,prd}_{aks,eks}`), Splunk O11y APM/detectors, instrumentação OTel via PR no gitops. Read-only nos clusters |
| `platform-engineer` | IDP: Backstage + Crossplane + ArgoCD. Desenha golden paths, escreve template/XRD/Composition (YAML), testa com `render`; código vai pro `idp-dev` |
| `idp-dev` | Dev sênior do IDP (time dev): plugins/módulos Backstage (TS), scaffolder actions, Crossplane functions (Go). Planeja antes, reusa upstream, testa. Sem merge, sem push em main |
| `project-manager` | Requisitos + gestão de tarefas (um agente, dois modos): spec com critérios de aceite, quebra por agente, cards PROJ, status. Sem código, nunca marca Done |

Sem `aws-specialist` (não portado): nenhum agent Claude Code consulta mais um subagent AWS read-only — perguntas de AWS ficam com quem já tinha o contexto (terraform-specialist, devops-agent, etc).

`mcpServers` em `claude/settings.json` só traz o que porta direto (`github-mcp`, `atlassian`, via `npx`). Os servers só-registro do Kiro (`eks-dev`, `eks-hml`, `eks-prd`, `awslabs.cloudwatch-mcp-server`, `awslabs.aws-documentation-mcp-server`, `awslabs.aws-network-mcp-server`, `splunk-mcp-combined`) ainda aparecem nas `tools:` de alguns agents mas não têm config de MCP server equivalente conhecida — configure a sua ou remova o `mcp__<nome>__*` correspondente.

Iniciar: copiar/symlink `claude/` para `~/.claude/` e usar o agent normalmente (Claude Code descobre `agents/` e `skills/` automaticamente).

## Todo lists

`devops-lead`, `dev-lead` e `project-manager` trabalham com todo list obrigatória (steering `todo-discipline.md`): item = resultado + dono + prova, gates viram itens, só marca feito com evidência verificada pelo lead. No Kiro CLI usa a tool `todo_list`; no Crew (sem a tool) usa arquivo em `~/work/__kiro-cli/temp/`.

## Hierarquia

```mermaid

flowchart TD
    %% Liderança DevOps
    subgraph OPS["DevOps Domain"]
        DOPS["devops-lead"]

        DOPS --> DA["devops-agent"]
        DOPS --> TS["terraform-specialist"]
        DOPS --> OS["observability-specialist"]
        DOPS --> PE["platform-engineer"]
        DOPS --> KT["k8s-troubleshooter"]
        DOPS --> LM["legacy-modernizer"]
        DOPS --> PMO["project-manager"]
    end

    %% Liderança de Desenvolvimento
    subgraph DEV["Development Domain"]
        DL["dev-lead"]

        DL --> DE["dev-agent"]
        DL --> IDP["idp-dev"]
        DL --> QA["qa-agent"]
        DL --> SR["semantic_reviewer"]
        DL --> RV["rules-validator"]
        DL --> PMD["project-manager"]
    end

    %% Comunicação entre domínios
    DOPS --> DL
    DOPS --> DE

    %% Fluxo de código vindo de operações
    OPS_CODE["Ops request"]
    OPS_CODE --> DOPS
    DOPS --> DL
    DL --> IDP

    %% Regra transversal AWS
    AWS["aws-specialist<br/>(AWS Read-only)"]

    DA -.-> AWS
    TS -.-> AWS
    OS -.-> AWS
    PE -.-> AWS
    KT -.-> AWS
    LM -.-> AWS
    DE -.-> AWS
    IDP -.-> AWS
    QA -.-> AWS
    SR -.-> AWS
    RV -.-> AWS
    PMO -.-> AWS
    PMD -.-> AWS

    %% Estilos
    classDef lead fill:#1d4ed8,color:#fff,stroke:#1e40af
    classDef agent fill:#334155,color:#fff,stroke:#475569
    classDef rule fill:#d97706,color:#fff,stroke:#b45309
    classDef external fill:#047857,color:#fff,stroke:#065f46

    class DOPS,DL lead
    class DA,TS,OS,PE,KT,LM,DE,IDP,QA,SR,RV,PMO,PMD agent
    class AWS rule
    class OPS_CODE external

```

> Engine v3: prompt vai inline no JSON (o v3 não lê `prompt: file://...`, log `Provider 'file' not found`) e o agent precisa de `permissions` (formato universal). Depois de editar um agent: `kiro-cli agent upgrade ~/.kiro/agents` e conferir que não apareceu deny `"*"`. `agent validate` não pega isso: teste abrindo uma sessão.
> Os arquivos de agent usam paths absolutos `/home/nathan/...` em `prompt` e `resources`. Ajustar se o usuário/home for outro.
> O `agent_config.json` antigo foi renomeado para `devops-agent.json` (o nome do agent vem do nome do arquivo).

## Uso

Symlink ou copiar os arquivos para os respectivos diretórios (`~/.copilot/`, `~/.kiro/`, `~/.claude/`).

> `mcp.json` / `claude/settings.json` têm token mascarado — configurar `GITHUB_PERSONAL_ACCESS_TOKEN` via env/secret, nunca commitar valor real.
