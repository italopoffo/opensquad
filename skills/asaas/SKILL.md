---
name: asaas
description: >
  Query and manage Asaas billing data — customers, charges (cobranças),
  subscriptions, splits, transfers, and webhooks — via Asaas's official
  MCP server. Use when a squad needs to read or act on real payment/billing
  data from an Asaas account.
description_pt-BR: >
  Consulte e gerencie dados de cobrança da Asaas — clientes, cobranças,
  assinaturas, splits, transferências e webhooks — via servidor MCP oficial
  da Asaas. Use quando um squad precisar ler ou agir sobre dados reais de
  pagamento/cobrança de uma conta Asaas.
description_es: >
  Consulta y gestiona datos de facturación de Asaas — clientes, cobros,
  suscripciones, splits, transferencias y webhooks — a través del servidor
  MCP oficial de Asaas.
type: mcp
version: "1.0.0"
mcp:
  server_name: asaas
  transport: http
  url: "https://docs.asaas.com/mcp"
  headers:
    access_token: ASAAS_MATRIZ_API_KEY
env:
  - ASAAS_MATRIZ_API_KEY
categories: [finance, automation, integration]
---

# Asaas — Billing & Payments Skill

## When to use

Use this skill to look up Asaas API schemas/docs and to test requests against
**sandbox only** — Asaas's own documentation confirms the interactive
`execute-request` tool only accepts sandbox calls, even when a production
key is supplied (a production key against `execute-request` returns
`401 invalid_access_token`, since the underlying host is always sandbox).

**For real production data (what the squad actually needs), do not use this
MCP's `execute-request`.** Instead, use the generic `web_fetch` skill (already
listed in `squad.yaml`) to call `https://api.asaas.com/v3/...` directly with
the `access_token` header — the same approach already implemented in
`squads/controle-financeiro-imobiliaria/app/src/lib/asaas/client.ts`. This MCP
remains valuable for discovering endpoint schemas/parameters
(`search-endpoints`, `get-endpoint`) before building that `web_fetch` call, and
for validating request shapes against sandbox during development.

## Multi-account setup

A single Asaas MCP connection is scoped to one account (one `access_token`).
This project may configure more than one Asaas account (e.g. matriz, filial).
Each additional account gets its own MCP server entry named `asaas_<account_id>`
(e.g. `asaas_matriz`, `asaas_filial_sp`), each pointing at
`https://docs.asaas.com/mcp` with its own `access_token` header.

Before calling any tool, check
`squads/controle-financeiro-imobiliaria/pipeline/data/contas-financeiras.md`
to resolve a business-facing account name (what the user typed at the scope
checkpoint) to the correct `mcp_server_name` to call.

## Status vocabulary — never blur these

Asaas charges move through a lifecycle. Getting this wrong corrupts every KPI
downstream:

- `PENDING` — awaiting payment, no money moved yet.
- `CONFIRMED` — payment confirmed but still reversible (chargeback/estorno risk).
- `RECEIVED` — the only status where the balance is definitively available.
- `OVERDUE` — past due date, unpaid.

Never treat `CONFIRMED` as equivalent to `RECEIVED`. If a report or dataset
needs "money actually available," filter to `RECEIVED` only.

## Instructions

1. Identify which account(s) are in scope and resolve each to its
   `mcp_server_name` via the account registry file above.
2. Use this MCP's discovery tools (`search-endpoints`, `get-endpoint`) to find
   the right endpoint, required parameters, and response schema — don't guess
   parameters.
3. To actually collect production data, call the resolved endpoint directly
   via `web_fetch` against `https://api.asaas.com/v3/<path>` with headers
   `access_token: <account's key>`, `Content-Type: application/json`, and a
   descriptive `User-Agent` (Asaas requires this for accounts created after
   2024-06-13). Only use this MCP's `execute-request` for sandbox testing —
   it will not reach production regardless of which key is supplied.
4. For listings, paginate (`offset`/`limit`, check `hasMore`) until the
   response signals no more pages — don't silently truncate a dataset (mirrors
   the pagination behavior already implemented in
   `squads/controle-financeiro-imobiliaria/app/src/lib/asaas/client.ts` for
   the read-only dashboard).
5. Record which account/environment (production vs sandbox) a result came
   from — never merge data across environments in the same dataset.
6. When a charge can't be linked to a known customer/property, surface it
   explicitly rather than dropping it silently.

## Available operations

- **List/search customers** — Asaas client records.
- **List/search charges (cobranças)** — status, due date, payment method, value.
- **Subscriptions** — recurring billing records.
- **Splits** — payment split configuration and history per charge.
- **Transfers** — outbound transfers from the Asaas balance.
- **Webhooks** — registered webhook endpoints and delivery history.
- **Schema/docs lookup** — fetch the OpenAPI schema for any endpoint before
  calling it, useful when unsure of required parameters.

## Setup

1. In the Asaas dashboard, generate an `access_token` for the account
   (Configurações > Integrações > API).
2. Set the corresponding env var (e.g. `ASAAS_MATRIZ_API_KEY`) so it can be
   resolved into the MCP server's `access_token` header.
3. Verify the connection by asking for a small read (e.g. "list the first
   page of charges") before relying on it in an automated pipeline run.
