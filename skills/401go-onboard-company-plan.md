---
name: Set up a new company and 401(k) plan in 401GO
description: Gather the affiliate firm, pooled plan, pricing tier and fund lineup ids a plan needs, create the company and 401(k) plan through the plan-setup endpoint, then verify the resulting plan provisions and match formulas.
api: openapi/401go-openapi-original.json
base_url: https://app.401go.com/api
operations:
  - affiliate_firms_list
  - affiliate_firms_affiliates_list
  - affiliate_firms_pooled_plans_list
  - affiliate_firms_pricing_tiers_list
  - affiliate_firms_fund_lineups_list
  - affiliates_pricing_tiers_list
  - plan_setup_create
  - plan_setup_update
  - companies_plan_provisions_retrieve
  - companies_matches_retrieve
  - companies_company_affiliates_retrieve
scopes:
  - affiliate_firm:read
  - affiliate:read
  - plan:write
  - plan:read
  - company:read
generated: '2026-08-02'
method: generated
source: https://developer.401go.com/reference/plan_setup_create
---

# Set up a new company and 401(k) plan in 401GO

`plan_setup_create` creates the company **and** the plan in one call. It is the most
constraint-heavy operation on the API — several fields accept opaque ids you must fetch first, and
the plan-type rules are enforced server-side. Resolve the ids before you build the payload.

## Step 1 — Resolve the opaque ids

Four `plan-setup` fields take ids that only other endpoints can give you:

| Field | Fetch it from |
|---|---|
| `acting_338` | `affiliate_firms_affiliates_list` (`GET /affiliate-firms/{affiliate_firm_id}/affiliates/`) or `affiliate_firms_list` |
| `pooled_plan` | `affiliate_firms_pooled_plans_list` (`GET /affiliate-firms/{affiliate_firm_id}/pooled-plans/`) |
| `billing_tier` | `affiliate_firms_pricing_tiers_list` or `affiliates_pricing_tiers_list` |
| `fund_lineup` | `affiliate_firms_fund_lineups_list` (`GET /affiliate-firms/{affiliate_firm_id}/fund-lineups/`) — 3(38) firms only |

Start from `affiliate_firms_list` (`GET /affiliate-firms/`) to get the firms your token can see.
All of these are paginated with `page` / `page_size`.

## Step 2 — Respect the field rules

**Mutually exclusive — provide at most one:**
- `pooled_plan` or `acting_338`
- `employer_match_tiers` or `non_elective_contribution`

**Both or neither:**
- `plan_type` and `plan_effective_date`
- `payroll_frequency` and `next_payroll_date`

**Dependencies:**
- `billing_tier` and `fund_lineup` each require `acting_338` **or** `pooled_plan`.
- `automatic_escalation_cap` requires a non-zero `automatic_enrollment_percentage`.
- `safe_harbor_exclude_hce_and_key` is valid only for safe harbor plan types.
- `vesting_schedule` requires `plan_type`.
- `grandfather_existing_employees` requires `plan_effective_date`.

## Step 3 — Get the match formula right for the plan type

The server enforces these exactly:

- **Basic Safe Harbor** — the standard formula only: 100% of the first 3%, then 50% of the next 2%
  (5% total). No variation.
- **Enhanced Safe Harbor** — exactly one match tier; total match a whole number between 4% and 6%.
- **Safe Harbor Non-Elective** — use `non_elective_contribution` (3–6%); do **not** send
  `employer_match_tiers`.
- **QACA Safe Harbor** — total match between 3.5% and 6%.
- **Traditional / Starter K / Solo K** — exactly one match tier; total match a whole number
  between 0% and 10%.

## Step 4 — Check the effective date and EIN

- `plan_effective_date` must generally be on or after the first of the following month. Exception:
  during December–February, Traditional, Solo K and Safe Harbor Non-Elective plans may use
  December 1st.
- Only **new** plans are supported. If the EIN already exists on the platform the request is
  rejected. Takeover plans must be set up outside the API — do not retry, escalate to
  partnersupport@401go.com.

## Step 5 — Create

Call `plan_setup_create` (`POST /plan-setup/`). The response returns an `object_id`. **That
`object_id` is the `company_id`** for every downstream call — participants, payroll, provisions,
matches. Persist it immediately.

## Step 6 — Update, if you must

Call `plan_setup_update` (`PUT /plan-setup/{company_id}/`).

- **PUT only.** PATCH partial updates are not supported; you must send a full replacement.
- Updates are **blocked** if the plan was modified through the 401GO web portal after the API
  setup, or if the plan is or was ever active. Design for create-correct rather than
  create-then-fix.

## Step 7 — Verify

- `companies_plan_provisions_retrieve` (`GET /companies/{company_id}/plan-provisions/`) — confirm
  eligibility, vesting, auto-enrollment and entry-date rules landed as intended. Note
  `exclude_highly_compensated_and_key_employees` is only present for safe harbor plans, and
  `hours_of_service` is only populated when the eligibility delay is 'Hours of Service'.
- `companies_matches_retrieve` (`GET /companies/{company_id}/matches/`) — confirm the match tiers.
- `companies_company_affiliates_retrieve` (`GET /companies/{company_id}/company-affiliates/`) —
  confirm the advisor relationship. Fields may be null; the `advisor` field is null when the
  company works with a firm but not a named advisor, and CRD numbers are not always available.

## Error handling

`plan_setup_create` returns **400** with field-keyed errors for every rule above. Each leaf is
`{"user_error_message": ..., "developer_error_detail": ...}` — `developer_error_detail` carries the
constraint text (for example `"cannot be more than 5 years if is_primary_residence is False"` in the
loan analogue). Surface `user_error_message` to the operator, log the developer detail.

`plan_setup_create` does **not** accept an `Idempotent-Key`. Because EIN reuse is rejected, a
duplicate submission fails safe rather than creating two plans — but check for an existing company
via `companies_list` before retrying a request whose response you did not see.
