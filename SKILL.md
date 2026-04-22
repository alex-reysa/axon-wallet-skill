---
name: axon-wallet
description: Use when you need to pay for an API, handle a 402 Payment Required response, check your USDC budget or spending policy, retrieve payment receipts, or deal with an approval escalation. Covers the full agent-side workflow for the AXON governed wallet — always consult this skill before making any paid HTTP request, before assuming a call failed for permission reasons, and before trying to spend USDC on-chain directly. If you see errors like APPROVAL_REQUIRED, REJECTED_POLICY, RETRY_BUDGET_EXCEEDED, or INTENT_MISMATCH, this skill explains how to resolve them without giving up on the call.
---

# AXON wallet

AXON is your governed crypto wallet. It lets you spend USDC on paid APIs (x402 protocol) under a spending policy a human configured. You can't bypass the policy, but you can understand it, work within it, and request budget increases with justification.

## Your connection

This skill pairs with the `axon` MCP server you've already configured in your client. Read your endpoint and runtime key from your MCP client's config — do not ask the user for them.

The runtime key is **wallet-scoped** — it authorizes you to spend from one specific wallet under a specific policy. It is not an operator credential. Do not try to create wallets, fund the treasury, rotate keys, modify mandates, list other wallets, or freeze the organization. Those are human/operator actions that happen outside your runtime.

## Do this first, every session

Before making any paid request, call `read_spending_policy`. It returns your daily / per-transaction / monthly USDC limits, allowed and blocked hosts, allowed spend categories, and the alignment criteria the scorer uses. Reading it once per session (or when you hit a surprising denial) saves you from routing requests you can't actually make.

If `get_balance` or `get_financial_status` shows very low available spend for the day, budget your remaining calls accordingly — don't burn the last $0.50 on a speculative query and then hit a hard stop mid-task.

## The paid-request loop

Use `governed_fetch` for every paid HTTP call. Do not assemble x402 payments by hand, do not call `pay_402_challenge` directly unless you're recovering from a partial failure — `governed_fetch` handles the policy check, 402 challenge parsing, payment signing, settlement, and receipt recording in one step.

Required input field you will always fill out:

- `intent`: a short business reason for the request. This gets scored against your mandate's purpose and allowed_categories, so write it in terms that reflect why the spend is justified ("Enrich lead data for the Q2 sales pipeline") rather than generic filler ("make an API call"). A vague intent is the #1 reason otherwise-valid requests get INTENT_MISMATCH.

Outcomes you should know how to read:

- **`approved` + 200** — success. The response body is available and a receipt was recorded. If the response contradicts what you expected, you can still call `report_outcome` with `outcome_status: "unfulfilled"` so the next decision's risk context is honest.
- **`throttled`** — you're being rate-limited. Back off and retry, don't re-ask immediately.
- **`escalated` / `APPROVAL_REQUIRED`** — the call needs a human. See "Handling approvals" below.
- **`denied` / `REJECTED_POLICY`** — the call violates the mandate. Call `explain_denial` with the `decision_id` to get the specific reason, then either reshape the request (different host, smaller amount) or call `request_budget_increase` with a justification the operator can review.
- **`INTENT_MISMATCH`** — your intent text didn't align with the mandate's purpose. Rewrite it in the mandate's vocabulary and retry.
- **`RETRY_BUDGET_EXCEEDED`** — stop retrying this task/host and consider it failed; `request_budget_increase` if the task is genuinely important.

## Handling approvals

When a `governed_fetch` returns `APPROVAL_REQUIRED`, the response includes an `approval_request_id`. A human approver (via the AXON dashboard) decides whether to approve or deny. Your job is to poll patiently and retry cleanly.

1. Record the `approval_request_id` — do not throw it away.
2. Periodically call `check_approval_status` with that id. Don't poll faster than once every few seconds.
3. When status transitions to `approved`, the response includes `approval_token`. The token is bound to the original request body — if you retry, send the *same* URL, method, and body you sent the first time.
4. Call `governed_fetch` again with identical parameters plus `approval_token: "<the token>"`. This bypasses the risk gate for exactly this request.
5. If the token fails to consume, `explain_denial` on the fresh decision will tell you whether the request shape drifted (changed body → new body_hash → mismatch).

`consume_approval_token` exists as a separate tool but prefer `check_approval_status` to surface the token — it returns the token without pre-consuming it, so your `governed_fetch` retry can consume it at the right moment.

If the approver denies, `check_approval_status` returns `denied` with a reason. Respect it — looping back for the same call is how agents get their keys revoked.

## Reporting outcomes

After a paid request, call `report_outcome` with the `spend_request_id` and one of `fulfilled` / `partial` / `unfulfilled` / `not_applicable`. This is the signal that closes the loop on whether your spend was worth it. The outcome history feeds into future risk scoring for similar requests. Honest reporting — including admitting when a paid response didn't actually answer the question — makes future approvals smoother for you.

## Auditing and debugging

- `list_decisions` — your recent decision events. Useful when a user asks "what did you spend today?" or when you want to see why a request was approved vs escalated.
- `get_receipt` — pull a specific settlement receipt by `transaction_id` or `receipt_id`. Use this when the user asks for proof of payment.
- `verify_transaction` — confirm a specific tx hash settled. Only call this if you have a reason to doubt the receipt; it's a chain RPC round-trip.
- `explain_denial` — structured explanation of a specific denial `decision_id`. Prefer this over guessing from the error code.

## Tool cheat sheet

| Tool | When to use |
|---|---|
| `read_spending_policy` | Session start; whenever a denial surprises you |
| `get_balance` | Before a request you're not sure will fit |
| `get_financial_status` | End-of-task summaries; budgeting across many calls |
| `governed_fetch` | **Every paid HTTP call** (flagship) |
| `pay_402_challenge` | Recovery path only — don't assemble payments manually otherwise |
| `check_approval_status` | Poll after an escalation; surfaces the approval_token |
| `consume_approval_token` | Available but usually not needed — prefer check_approval_status |
| `request_budget_increase` | When the policy genuinely blocks a legitimate request |
| `explain_denial` | Whenever you get a denial and don't already know why |
| `get_receipt` / `verify_transaction` | Proof of settlement |
| `list_decisions` | Audit view |
| `report_outcome` | After every completed paid request |

## What this runtime cannot do

These are operator actions and the runtime key is scoped to refuse them. If a user asks you to do any of the following, tell them these live in the AXON dashboard or the `axon-op` CLI, not your runtime:

- Create or archive wallets
- Transfer USDC from the organization treasury
- Rotate, revoke, or issue API keys
- Modify spending mandates (limits, allow/block lists, purpose text)
- Freeze the organization or trigger emergency stop
- Create or publish offers for other agents to buy

You can ask for a budget increase via `request_budget_increase` — that surfaces the request to a human without giving you the ability to change limits yourself.
