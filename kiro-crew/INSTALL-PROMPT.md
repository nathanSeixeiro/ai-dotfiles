# INSTALL-PROMPT

Cole o bloco abaixo no seu **Kiro CLI** (na máquina onde quer instalar os agents). Antes, confirme seu usuário Windows e troque `<WINUSER>` por ele (ex.: `SN1091308`).

---

Instale os agents de DevOps MyOrg neste Kiro, a partir da pasta `kiro-crew/` deste repositório clonado. Contexto da máquina:

- SO: Windows. Kiro Crew (dashboard).
- `gh` CLI está instalado no **Windows** — use direto pra GitHub (push, PR, branch, workflow run).
- `kubectl`, `az`, `terraform`, `aws` estão no **WSL** — chame como `wsl bash -lc "<comando>"`.
- Clones de repositório podem ficar no Windows ou no WSL, tanto faz.
- Meu usuário Windows é `<WINUSER>`, então meu home Kiro é `C:\Users\<WINUSER>\.kiro\`.

Faça, nesta ordem:

1. Copie os arquivos de apoio da pasta `kiro-crew/` deste repo para o meu home Kiro:
   - `kiro-crew/prompts/*.md`   -> `C:\Users\<WINUSER>\.kiro\prompts\`
   - `kiro-crew/knowledge/*`    -> `C:\Users\<WINUSER>\.kiro\knowledge\` (mantendo as subpastas por agent)
   - `kiro-crew/steering/*.md`  -> `C:\Users\<WINUSER>\.kiro\steering\`

2. Para cada agent (`devops-lead`, `devops-agent`, `k8s-troubleshooter`, `legacy-modernizer`, `terraform-specialist`, `observability-specialist`, `platform-engineer`, `project-manager`):
   - pegue o `.json` em `kiro-crew/agents/<nome>.json`;
   - substitua todo `<WINUSER>` pelo meu usuário real;
   - crie o agent em `C:\Users\<WINUSER>\.kiro\agents\<nome>.json`.
   - Se `.kiro\agents\` for path protegido e você não conseguir escrever, me entregue cada `.json` final e eu salvo/uso `kiro-cli agent create`.

   - Não troque `prompt` por `file://`: o prompt já vem inline (engine v3 não lê `file://`). Não apague o bloco `permissions`.

3. Confirme: liste os agents (`kiro-cli agent list`) e valide cada um (`kiro-cli agent validate <nome>`). Se algum resource/prompt apontar pra arquivo que não existe, me diga qual.

Regras que NÃO podem ser afrouxadas nos agents (já vêm no JSON, só não quebre):
- `k8s-troubleshooter` e `devops-agent`: nenhuma mutação em prd (`aks-prd`, `eks-prod`); sem `kubectl delete`.
- `k8s-troubleshooter`: `az keyvault secret show` negado (só `list`, chaves).
- `terraform-specialist`: nunca `terraform plan/apply/destroy/import/state`; só `init -backend=false`, `fmt`, `validate`.
- `observability-specialist`: kubectl só leitura; nada de `get secret`.
- `project-manager`: write só em `work\__kiro-cli\**`; nunca marca Done no Jira.
- `legacy-modernizer`: write só em `work\**`; sem `terraform apply/plan`, sem `gh secret/variable set`, sem `gh pr merge`, sem push em main.
- Todos: sem `rm -rf`, sem `sudo`, sem force-push, sem `git reset --hard`.

No fim, me diga em 2 linhas: o que instalou e o que não deu pra validar.
