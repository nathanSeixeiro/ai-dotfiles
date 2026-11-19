# Habblaud — prompt de instalação (corrigido)

Prompt de instalação do Habblaud, versão corrigida com suporte ao **Kiro CLI**.
Vale igual no Claude Code, Codex e Kiro CLI. Repo: https://github.com/marmottajr/habblaud

- **Pasta:** `~/habblaud` (no Windows, dentro do WSL)
- **Endereço final:** https://habblaud.localhost
- **Suporte ao Kiro:** PR #129 (branch `feat/kiro-source` do fork `gihovani/habblaud`) até entrar no `main`.

## Fluxo

1. **Ferramentas:** `git`, Node 22.12+. Docker rodando (Docker Desktop no Windows com WSL integration). Agentes: conecte só os que existirem (Claude `~/.claude*`, Codex `~/.codex*`, Kiro `~/.kiro`).
2. **Baixar:** `git clone https://github.com/marmottajr/habblaud.git ~/habblaud`. Se uso Kiro e falta `~/habblaud/mod/habblaud-kiro`, usar `git clone -b feat/kiro-source https://github.com/gihovani/habblaud.git ~/habblaud`. Depois `npm install`.
3. **Subir:** `npm run docker:up` (ou nativo — ver nota da mãozinha).
4. **Claude:** `npm run mod:install` + `npm run mod:status`.
5. **Codex:** `npm run codex:install` + `npm run codex:status` + serviço `npm run codex:bridge`.
6. **Kiro:** `npm run kiro:install` + `npm run kiro:status`. Windows nativo: `node install.mjs install` via PowerShell contra `\\wsl.localhost\<distro>\...`.
7. **Portless:** `npm install -g portless` + `portless alias habblaud 4747`; proxy no login + confiar no certificado.
8. **Testar:** `http://localhost:4747/api/health`, `npm run kiro:status`, `https://habblaud.localhost`.

## Notas deste ambiente (Windows + WSL2)

- **Docker no WSL:** se `docker` não aparece no WSL → Docker Desktop → Settings → Resources → WSL integration → ligar distro. Se o usuário já está no grupo `docker` mas a sessão WSL é antiga, use `sg docker -c "<cmd>"` em vez de reiniciar/sudo.
- **Mãozinha de aprovação do Kiro:** só funciona com o Habblaud rodando **nativo no WSL** (lê `/proc`). NÃO funciona no Docker (PIDs do host invisíveis no container) nem no Kiro nativo do Windows (detectado por `tasklist`, sem o sinal de "parado esperando aprovação"). Para ter a mãozinha: `npm run build` + `node dist/server/index.js` via serviço systemd de usuário, usando o Kiro dentro do WSL.
- **Rodar nativo (sem Docker):** serviço systemd de usuário em `~/.config/systemd/user/habblaud.service` apontando `node dist/server/index.js`, `Restart=always`, `enable --now`; `loginctl enable-linger <user>` para subir no boot.
- **Kiro 2.x vs 3.x:** 2.x grava o hook em cada `~/.kiro/agents/*.json` (com backup); 3.x grava `~/.kiro/hooks/habblaud.json`.
- Reabrir as sessões do Kiro depois de instalar o hook (só vale para sessões novas).
