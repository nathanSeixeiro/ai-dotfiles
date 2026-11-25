---
name: enterprise-report
description: Create a corporate HTML report (my-org layout, A4, print-ready) and export it to PDF. Use when the user asks for a report, relatório, RCA, incident/status report, analysis to send to devs/management, or to export something to HTML/PDF.
---

# Enterprise Report (HTML + PDF)

Corporate report for devs, managers or third parties. One HTML file with inline CSS, prints clean to A4 PDF.

## Steps

1. **Evidence first.** Gather real data before writing: logs, `kubectl`, `gh pr view`, Jira card. Never assume a PR/card state. Every claim in the report must come from something you checked.
2. **Split by owner.** Classify each problem: infra (DevOps) vs app/data (Dev). If unsure, say "em análise" and what check will decide it. Don't send devs errors that infra may cause.
3. **Copy the template** `~/.claude/skills/enterprise-report/template.html` to `~/work/__kiro-cli/reports/<topic>.html` (kebab-case, no date in name). Fill the placeholders `{{...}}`. Delete sections that don't apply.
4. **Language = audience.** Devs/Jira at MyOrg: Portuguese (pt-BR). Ask if unclear.
5. **Export PDF** (Windows Chrome from WSL, no install):
   ```bash
   cd ~/work/__kiro-cli/reports
   IN=$(wslpath -w "$PWD/<topic>.html"); OUT=$(wslpath -w "$PWD/<topic>.pdf")
   "/mnt/c/Program Files/Google/Chrome/Application/chrome.exe" --headless=new --disable-gpu --no-pdf-header-footer --print-to-pdf="$OUT" "$IN"
   ```
   Edge fallback: `/mnt/c/Program Files (x86)/Microsoft/Edge/Application/msedge.exe` with same flags.
6. **Check the layout.** Screenshot to temp, read it, fix and re-export if anything breaks. Then delete the screenshot:
   ```bash
   "/mnt/c/Program Files/Google/Chrome/Application/chrome.exe" --headless=new --hide-scrollbars --window-size=900,1300 --screenshot="$(wslpath -w ~/work/__kiro-cli/temp/preview.png)" "$IN"
   ```
7. **Link it.** If there is a PROJ card, add a step comment saying the report is ready (the user sends it, don't claim it was sent).
8. **Tell the user** the HTML and PDF paths. **Always end the answer** with the browser address to paste in Chrome/Edge, built from the real distro name (`$WSL_DISTRO_NAME`) and absolute path:
   ```
   file://wsl.localhost/<distro>/home/<user>/work/__kiro-cli/reports/<topic>.html
   ```
   Example: `file://wsl.localhost/Ubuntu-24.04/home/nathan/work/__kiro-cli/reports/<topic>.html`. Also add the PDF address the same way.

## Content rules

- Sections (keep order, drop unused): Resumo · Status · Infra (DevOps) · Aplicação (Dev) · Próximos passos (ação, responsável, status) · Referências.
- Status tags: `ok` (Resolvido/OK), `warn` (PR aberto / Em análise / Pendente), `err` (Erro / Ação dev).
- Name the mechanism and the number: error message, count, limit, timestamps (UTC), versions, PR links.
- Error excerpts in `<pre>`, trimmed. Group repeated errors with counts in a table.
- **Never** include secret values, connection strings, passwords, tokens. IDs only when needed.
- Escape `<`, `>`, `&` inside HTML text (`&lt;` `&gt;` `&amp;`).
- Short sentences. No filler. One idea per bullet.
- Footer: "Documento gerado pelo time de DevOps. Não contém valores de segredos."

## Other export options (if the user asks)

- Manual: open in Chrome, Ctrl+P, "Save as PDF", untick "Headers and footers".
- Word with corporate template: `pandoc r.md -o r.docx --reference-doc=<template>.docx` (pandoc not installed by default).
- Confluence page via Atlassian MCP (exports to PDF with company layout). Ask before publishing.
