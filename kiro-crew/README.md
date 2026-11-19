# kiro-crew

Agents adaptados para **Kiro Crew** (dashboard) rodando em **Windows**, a partir dos agents originais do Kiro CLI em `../kiro/`.

## O que mudou em relação a `../kiro/agents/`

| Mudança | Motivo |
|---|---|
| Nomes de tool `read`/`write`/`shell` -> `fs_read`/`fs_write`/`execute_bash` | Nomes nativos do Crew (os aliases funcionam, mas o Crew usa estes) |
| `"model": "auto"` | Padrão do Crew (modelo escolhido no servidor) |
| MCP de registry removidos (eks-dev, eks-hml, cloudwatch, github-mcp) | Não existem no Crew pessoal; evitam tool fantasma. Acesso agora é via `gh`/`kubectl`/`az` no shell |
| `gh` roda no **Windows** direto | `gh` foi instalado no Windows |
| `kubectl`/`az`/`terraform`/`aws` via `wsl bash -lc "..."` | Essas ferramentas estão no WSL |
| Paths `/home/nathan/...` -> `C:/Users/<WINUSER>/.kiro/...` | Home do Windows; troque `<WINUSER>` pelo seu usuário |
| `fs_write.allowedPaths` aponta para `work\**` no Windows e WSL | Saída de arquivos |

As regras de segurança (deny de mutação em prd, deny de `az keyvault secret show` no troubleshooter, deny de `terraform apply`, `rm -rf`, force-push) foram **preservadas**.

## Estrutura

```
kiro-crew/
  agents/        devops-lead, devops-agent, k8s-troubleshooter, legacy-modernizer, terraform-specialist, observability-specialist, platform-engineer, project-manager (.json)
  prompts/       <agent>.md (um por agent, exceto devops-lead)
  knowledge/     <agent>/*.md  (docs de referência)
  steering/      caveman-mode.md, work-rules.md, app-onboarding.md, todo-discipline.md, jira-cards.md, workflow-path-triggers.md, devops-lead-playbook.md
  INSTALL-PROMPT.md   cole no Kiro CLI pra criar tudo local
```

## Instalar

1. Troque `<WINUSER>` em todos os `.json` de `agents/` pelo seu usuário Windows (ex.: `SN1091308`).
2. Copie:
   - `agents/*.json`   -> `C:\Users\<WINUSER>\.kiro\agents\`
   - `prompts/*.md`    -> `C:\Users\<WINUSER>\.kiro\prompts\`
   - `knowledge/*`     -> `C:\Users\<WINUSER>\.kiro\knowledge\`
   - `steering/*.md`   -> `C:\Users\<WINUSER>\.kiro\steering\`
3. Confira com `kiro-cli agent list` e `kiro-cli agent validate <nome>`.

> `.kiro\agents\` é path protegido no Crew — o agent não escreve nele; você copia/cria manualmente ou via `kiro-cli agent create`.

Ou use `INSTALL-PROMPT.md`: cole no Kiro CLI e ele cria os arquivos pra você.
