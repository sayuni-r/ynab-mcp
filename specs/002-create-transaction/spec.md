# Spec 002: Create a transaction

- **Status:** Draft
- **Date:** 2026-11-17
- **Depends on:** spec 001 (the YNAB client, the money conversion and how a plan is chosen)

## Goal

Let me log a transaction by telling Claude, with a confirmation step so nothing is written by mistake.

## User stories

- **US1.** As a YNAB user, I can say "Log $12.50 at Starbucks under Dining from Checking", and it is recorded in YNAB once I confirm.
- **US2.** As a YNAB user, I see exactly what will be written before it is written.

## Functional requirements

- **FR-1 `create_transaction`.** Takes a plan (optional, same rules as spec 001), account, amount, payee name, and optionally a category, date and memo.
    - Amount is in currency units. Negative means money out, positive means money in.
    - With no date, the default is today in my local time zone.
- **FR-2 Confirm step.** A call without `confirm=true` writes nothing. It returns a preview of the resolved account, category, amount, payee and date. Only a call with `confirm=true` writes the transaction.
- **FR-3 Resolving names.** Account and category names match without regard to case. If a name matches nothing, or more than one item, the tool returns an error listing the candidates and writes nothing. It never guesses.
- **FR-4 Money.** The amount is converted to milliunits exactly (`-12.50` becomes `-12500`). An amount with more decimal places than the plan's currency allows is rejected.
- **FR-5 Result.** After writing, the tool returns the new transaction's id and the values YNAB stored.
- **FR-6 Read-only switch.** If `YNAB_READ_ONLY=true`, this tool is not registered at all.

## Acceptance criteria

| ID | Given | When | Then |
| --- | --- | --- | --- |
| AC1 | valid inputs and no `confirm` | `create_transaction` is called | nothing is written and a preview is returned |
| AC2 | an amount of `-12.50` | the transaction is confirmed | YNAB stores `-12500` milliunits |
| AC3 | an account name that matches two accounts | `create_transaction` is called | the error lists both accounts and nothing is written |
| AC4 | a category name that matches nothing | `create_transaction` is called | the error says so and nothing is written |
| AC5 | no date | the transaction is confirmed | its date is today in my local time zone |
| AC6 | an amount of `12.505` in a USD plan | `create_transaction` is called | it is rejected |
| AC7 | `YNAB_READ_ONLY=true` | the server starts | `create_transaction` is not in the tool list |
| AC8 | valid inputs and `confirm=true` | the transaction is written | it shows up in the YNAB app with the same values (manual check) |

## Out of scope

- Editing or deleting transactions
- Split transactions
- Transfers between accounts
- Scheduled transactions

## Open questions

- Confirm step: a `confirm` parameter, or MCP elicitation (the client asks me directly)?
- Should new transactions be marked approved, or left for me to approve in the app?
- If the model retries after a timeout, how do we avoid creating the same transaction twice?