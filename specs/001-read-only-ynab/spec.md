# Spec 001: Read-only YNAB MCP server

- **Status:** Draft
- **Date:** 2026-11-17
- **Depends on:** nothing

## Goal

Let an AI assistant answer questions about my YNAB plan, such as balances and recent spending, without being able to change anything.

## Background

YNAB has a REST API (`api.ynab.com/v1`) that exposes plans (formerly called budgets), accounts and transactions. This server exposes a small, read-only subset of it as MCP tools so Claude Code or Claude Desktop can query it in conversation.

## User stories

- **US1.** As a YNAB user, I can ask Claude which plans I have, so I can choose the one to ask about.
- **US2.** As a YNAB user, I can ask Claude for my account balances.
- **US3.** As a YNAB user, I can ask Claude what I spent over a period, e.g. "What did I spend on dining this month?"

## Functional requirements

- **FR-1 `list_plans`.** Returns every plan in the account with its id, name and last-modified date.
- **FR-2 `get_accounts`.** Takes a plan. Returns that plan's open accounts with id, name, type, balance and cleared balance. Closed and deleted accounts are left out.
- **FR-3 `get_transactions`.** Takes a plan, an optional `since_date` (YYYY-MM-DD) and an optional account. Returns transactions with date, amount, payee, category, account, memo and cleared status, newest first.
    - With no `since_date`, the default is the first day of the current month.
    - Deleted transactions are never returned.
    - At most 200 transactions are returned. If more match, the response says it was truncated.
- **FR-4 Money.** Every amount is returned in currency units, not milliunits (YNAB stores $12.34 as `12340`). Responses include the plan's currency code.
- **FR-5 Choosing a plan.** If a tool's plan is omitted, the server uses `YNAB_PLAN_ID`. If that is not set either, the tool returns an error telling the model to call `list_plans` first.
- **FR-6 Errors.** An invalid token, an unknown plan, a rate limit or a network failure each return a short, clear error message. No stack traces.
- **FR-7 Read-only.** No tool can create, update or delete anything in YNAB.

## Non-functional requirements

- **NFR-1.** Runs locally over stdio, launched by Claude Code or Claude Desktop.
- **NFR-2.** The token comes from the `YNAB_API_TOKEN` environment variable. It never appears in tool output, logs or git.
- **NFR-3.** YNAB allows about 200 requests per hour per token, so each tool call makes at most two API requests.
- **NFR-4.** Tool output contains only the fields listed above, to keep it small for the model.

## Acceptance criteria

| ID | Given | When | Then |
| --- | --- | --- | --- |
| AC1 | a valid token | `list_plans` is called | every plan is returned with its id and name |
| AC2 | an account with a balance of `1234560` milliunits | `get_accounts` is called | its balance is shown as `1234.56` |
| AC3 | a transaction of `-12340` milliunits | it is returned | its amount is `-12.34` |
| AC4 | a closed account | `get_accounts` is called | it is not listed |
| AC5 | `since_date` = 2026-11-01 | `get_transactions` is called | nothing dated before 2026-11-01 is returned |
| AC6 | a deleted transaction | `get_transactions` is called | it is not returned |
| AC7 | more than 200 matching transactions | `get_transactions` is called | the 200 newest are returned and the response says it was truncated |
| AC8 | an invalid token | any tool is called | the error says the token is invalid and does not contain the token |
| AC9 | no plan given and no `YNAB_PLAN_ID` | `get_accounts` is called | the error tells the model to call `list_plans` first |
| AC10 | the server is connected to Claude Code | I ask "What did I spend on dining this month?" | Claude calls `get_transactions` and the total matches the YNAB app (manual check) |

## Out of scope

- Any write: creating, editing or deleting transactions (see spec 002)
- OAuth, HTTP transport and hosting
- Caching and delta sync
- Scheduled transactions, monthly category budgets, payees as their own tool
- More than one YNAB user

## Open questions

- Should `get_transactions` also filter by category on the server, or is it fine to leave that to the model?
- Default window: the current month, or the last 30 days?
- Split transactions: return the parent only, or each subtransaction with its own category?
- Transfers between my own accounts: include them, or hide them so they don't count as spending?