---
name: gmud-create
description: Create a GMUD (Mudança Normal) in Jira Service Management project GMUD at example.atlassian.net. Use when the user asks to create, open, draft or register a GMUD / mudança / change request, or to prepare its fields. Always asks the variable fields, shows the payload and only creates after explicit user OK.
---

# GMUD Create (Mudança Normal)

Creates a GMUD via Atlassian MCP. Scope: request type **Mudança Normal** only.
Emergencial (334) and Pré-Aprovada (336) are out of scope: tell the user, do not guess their fields.

## Fixed IDs (verified read-only against GMUD-879)
- cloudId: `example.atlassian.net`
- serviceDeskId: `22` (project GMUD)
- requestTypeId: `335` (Mudança Normal)
- Create tool: `executeWrite` -> `createJsmRequest` (load `tool::atlassian-mcp::executeWrite` first)
- Read tools: `executeRead` -> `getJsmRequestTypeFields`, `getJsmRequest`

## Step 1 - Ask (never assume these)
Ask in ONE message, only what the user did not already give:
1. **Resumo** (title)
2. **Início e fim da janela** (date + time, America/Sao_Paulo)
3. **Gera indisponibilidade?** If yes: início e fim da indisponibilidade (default = same as window, confirm)
4. **É reversível?**
5. **Roda no horário de funcionamento do serviço de TI?**
6. **O que muda** (free text, to build the impact description)

## Step 2 - Defaults (use when user gives no other value; show them in the preview)
| Field | ID | Default |
|-------|----|---------|
| Responsável pela Alteração | `customfield_10339` | <requester name> (`<REQUESTER_ACCOUNT_ID>`) |
| Responsável por Validar | `customfield_10341` | <requester> (same accountId) |
| Responsáveis Aprovadores | `customfield_10076` | <approver name> (`<APPROVER_ACCOUNT_ID>`) |
| Origem da Alteração | `customfield_10340` | Melhoria |
| Serviço de acesso público? | `customfield_10188` | Não |
| Rotina crítica de negócio? | `customfield_10191` | Não |
| Impacta SIG/Legados? | `customfield_10192` | Não |
| Impacta serviço de TI crítico? | `customfield_10185` | Não |

Ask again if the change clearly touches a public service, a critical service or SIG/legacy: the default Não would be wrong.

## Step 3 - Build the description
- Field `customfield_12915` (Impacto Gerado pela Mudança), textarea.
- Plain text, bullet list with `* `, one line per impact. Style of GMUD-879:
  - what freezes / what is lost during the window
  - URL, access or integration changes
  - what the team must redo after
  - what becomes read-only
- Write only what the user said. Never invent numbers or impacts. Mark doubts as `[confirmar]`.
- Never put secret values, tokens or personal data.

## Step 4 - Validate before preview
- Re-read the form: `getJsmRequestTypeFields(serviceDeskId=22, requestTypeId=335)`. If a required field is missing from this skill, stop and ask.
- Fim > Início. Indisponibilidade inside the window.
- Início in the past -> warn: "janela retroativa", ask explicit confirmation. (Field Mudança Retroativa is not on the create form.)
- Datetime format: `YYYY-MM-DDTHH:mm:00.000-0300`.
- Radio/select values go as `{"value":"Sim"}` (label, per `usageNotes` of the form). Radio option IDs for reference: Sim/Não public 10167/10168, horário 10169/10170, indisponibilidade 10171/10172, rotina crítica 10173/10174, SIG 10175/10176, crítico 10161/10162, reversível 10788/10789, Origem Melhoria 10794.

## Step 5 - Preview and confirm (mandatory)
Show a table: field, value, source (user / default). Show the payload. Ask: "Criar? (sim/não)".
Create only after an explicit yes. No yes = create nothing. Never create in the same turn as the preview.

## Step 6 - Create
```
executeWrite({
  name: "createJsmRequest",
  cloudId: "example.atlassian.net",
  inputs: {
    serviceDeskId: "22",
    requestTypeId: "335",
    requestFieldValues: {
      summary: "<Resumo>",
      customfield_10063: "<início>",
      customfield_10064: "<fim>",
      customfield_10188: {"value": "Não"},
      customfield_10189: {"value": "<Sim|Não>"},
      customfield_10190: {"value": "<Sim|Não>"},
      customfield_10191: {"value": "Não"},
      customfield_10192: {"value": "Não"},
      customfield_10185: {"value": "Não"},
      customfield_10337: {"value": "<Sim|Não>"},
      customfield_10340: {"value": "Melhoria"},
      customfield_10076: [{"accountId": "<APPROVER_ACCOUNT_ID>"}],
      customfield_10341: {"accountId": "<REQUESTER_ACCOUNT_ID>"},
      customfield_10339: {"accountId": "<REQUESTER_ACCOUNT_ID>"},
      customfield_12915: "<descrição>",
      customfield_10383: "<início indisp, só se gera indisponibilidade>",
      customfield_10384: "<fim indisp, só se gera indisponibilidade>"
    }
  }
})
```
Omit the two indisponibilidade fields when there is no outage.
On a validation error: fix once from the error text, show the change, ask again before retrying.

## Step 7 - Verify and report
- `getJsmRequest(issueIdOrKey=<key>)`: confirm status `Aberto` and the fields match the preview.
- Reply: key, link `https://example.atlassian.net/browse/<key>`, status, anything that differs.

## Limits (not covered)
- CMDB field `customfield_12255` (sistema impactado): left empty. Not tested.
- Not covered: Risco, Impacto, Plano de Retorno/Testes/Comunicação. They are edit-screen fields, not on the create form.
- Does not link or comment on PROJ cards. Does not move status. Never approves (`decideJsmRequestApproval` is off limits).
- Never mark or transition the GMUD. Approval stays with the approvers.
