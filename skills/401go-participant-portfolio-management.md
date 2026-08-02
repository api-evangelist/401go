---
name: Manage a participant's 401GO investment portfolio
description: Read a participant's available investment options, current holdings and rebalance settings, set target allocations correctly (including the 0% trap), and trigger or cancel a rebalance inside a trading window.
api: openapi/401go-openapi-original.json
base_url: https://app.401go.com/api
operations:
  - participants_investment_options_list
  - companies_investment_options_list
  - investments_retrieve
  - participants_portfolio_list
  - participants_portfolio_create
  - participants_portfolio_settings_retrieve
  - participants_portfolio_settings_create
  - participants_start_participant_rebalance_create
  - participants_cancel_participant_rebalance_create
  - participants_investment_history_list
  - participants_investment_performance_list
  - participants_advisor_models_retrieve
  - participants_advisor_models_create
scopes:
  - participant:read
  - participant:write
generated: '2026-08-02'
method: generated
source: https://developer.401go.com/reference/participants_portfolio_create
---

# Manage a participant's 401GO investment portfolio

Two rules here are non-obvious and will cost a participant real money if you get them wrong: a **0%
allocation does not remove a holding**, and a **POST of allocations replaces the entire prior
allocation set**. Read the allocation section before writing any code that posts a portfolio.

## Step 1 — Check the participant may self-direct

`participants_portfolio_settings_retrieve`
(`GET /participants/{participant_id}/portfolio-settings/`) returns:

- whether the participant can self-direct,
- `auto_rebalance` and the rebalance percent threshold,
- `can_cancel_rebalance` — **true only while you are inside a trading window**,
- `next_trading_window` — the start of the next window.

Treat `next_trading_window` with caution. Leave roughly a **30-minute buffer** so a trade can
actually complete inside the window.

The plan itself must permit it: `companies_plan_provisions_retrieve` exposes
`allow_self_direct_for_participants`.

## Step 2 — Get the eligible investments

- `participants_investment_options_list`
  (`GET /participants/{participant_id}/investment-options/`) — what this participant may hold.
  Optionally paginate with `page` / `page_size`.
- `companies_investment_options_list` (`GET /companies/{company_id}/investment-options/`) — the
  plan-level lineup.
- `investments_retrieve` (`GET /investments/{investment_id}/`) — detail for one fund.

Only allocate to investments returned by the participant-level list.

## Step 3 — Read the current portfolio

`participants_portfolio_list` (`GET /participants/{participant_id}/portfolio/`) returns the holdings
with a state per investment:

- `ACTIVE` — part of the portfolio.
- `PENDING_SALE` — removed from the portfolio and being fully liquidated.

A `PENDING_SALE` holding is on its way out; do not re-add it mid-liquidation expecting the sale to
stop.

## Step 4 — Set target allocations

`participants_portfolio_create` (`POST /participants/{participant_id}/portfolio/`) expects:

- `investments` — a list of investment ids
- `weights` — a list of target percent weights, **aligned by index** with `investments`

Two traps:

1. **A POST replaces the prior allocation entirely.** Whatever you send *is* the allocation. Always
   send the complete intended target set, never a delta.

2. **Setting an allocation to 0% does not remove the investment.** It stays in the portfolio. If
   `auto_rebalance` is disabled, *no trades happen at all* until a manual rebalance is triggered —
   so a 0% weight can sit there doing nothing indefinitely.

   **To remove an investment, omit it from the POST.** Once omitted, the system sells **all** shares
   of that investment **regardless of whether `auto_rebalance` is enabled**, and the holding enters
   `PENDING_SALE` until liquidation completes.

That asymmetry is the important part: omission liquidates unconditionally, 0% may do nothing.

## Step 5 — Rebalance settings

`participants_portfolio_settings_create`
(`POST /participants/{participant_id}/portfolio-settings/`) updates the auto-rebalance flag, the
rebalance percent, and the participant's self-direct opt-in.

## Step 6 — Trigger or cancel a rebalance

- `participants_start_participant_rebalance_create`
  (`POST /participants/{participant_id}/start-participant-rebalance/`)
- `participants_cancel_participant_rebalance_create`
  (`POST /participants/{participant_id}/cancel-participant-rebalance/`)

Cancel only works while `can_cancel_rebalance` is true — i.e. inside the trading window. Re-read
`participants_portfolio_settings_retrieve` immediately before attempting a cancel rather than
trusting a cached value.

## Step 7 — Advisor models

- `participants_advisor_models_retrieve` (`GET /participants/{participant_id}/advisor-models/`)
- `participants_advisor_models_create` (`POST /participants/{participant_id}/advisor-models/`)

Selecting an advisor model is the managed alternative to hand-setting weights. Do not drive both
paths against the same participant in the same cycle.

## Step 8 — Confirm what happened

- `participants_investment_history_list`
  (`GET /participants/{participant_id}/investment-history/`) — transactions, optionally paginated.
  Statuses are `pending`, `confirmed`, `failed`, `dividend`. **Poll for `confirmed`; a trade is not
  done when the POST returns 2xx.**
- `participants_investment_performance_list`
  (`GET /participants/{participant_id}/investment-performance/`) — one portfolio snapshot per day
  between `start_date` and `end_date`.

## Error handling

Errors are 401GO's custom envelope — `{"user_error_message", "developer_error_detail"}` — not
RFC 9457, and field errors nest to match your submitted JSON.

- **400** — weights do not sum correctly, an investment is not available to this participant, or
  `investments` and `weights` are misaligned in length.
- **401** — expired access token (60 minutes).
- **403** — missing `participant:write`, or the endpoint+method is not on your client's allow list.

**No `Idempotent-Key` is accepted on any portfolio operation.** Do not blind-retry a portfolio POST
whose response you did not observe — re-read `participants_portfolio_list` first and compare against
your intended target set.
