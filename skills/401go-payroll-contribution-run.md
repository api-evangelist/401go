---
name: Run a 401GO payroll contribution cycle
description: For each pay period, read each participant's current deferral elections and active loan payments from 401GO, calculate the deductions against gross pay, and submit the payroll contribution file back to 401GO idempotently.
api: openapi/401go-openapi-original.json
base_url: https://app.401go.com/api
operations:
  - companies_list
  - companies_plan_provisions_retrieve
  - companies_matches_retrieve
  - companies_participants_list
  - participants_deferrals_retrieve
  - companies_submit_payroll_create
  - participants_payroll_lines_list
scopes:
  - company:read
  - company:write
  - participant:read
generated: '2026-08-02'
method: generated
source: https://developer.401go.com/docs/payroll-integration
---

# Run a 401GO payroll contribution cycle

This is 401GO's marquee partner flow — the "360 degree" payroll integration. Every pay period you
pull the current deduction instructions out of 401GO, apply them to the payroll you just ran, and
push a contribution file back.

## Before you start

- Authenticate with OAuth 2.0. Send `Authorization: Bearer <access_token>`. Access tokens live 60
  minutes; refresh with `grant_type=refresh_token` at `POST https://app.401go.com/api/o/token`.
- Your client must be on 401GO's endpoint + HTTP-method allow list for every operation below. A
  correctly scoped token still returns **403** for an operation you were not granted. Request access
  at <https://developer.401go.com/docs/api-endpoint-and-method-access>.
- Optionally pin behaviour with the `api-version` header (an ISO `YYYY-MM-DD` date). Omitting it
  pins you to the version current when your application was registered.

## Steps

1. **Find the companies you can act for.** Call `companies_list`
   (`GET /companies/`). Paginate with `page` and `page_size`; read `count`, `next`, `previous`,
   `results`. Each company has an `object_id` — that is the `company_id` for everything below.
   Skip companies whose `status` is `SETUP_PENDING`.

2. **Cache the plan rules once, not per payroll.** Call `companies_plan_provisions_retrieve`
   (`GET /companies/{company_id}/plan-provisions/`) for eligibility, vesting, auto-enrollment
   percentages and `loans_permitted`, and `companies_matches_retrieve`
   (`GET /companies/{company_id}/matches/`) for `plan_matches`, `discretionary_matches` and the
   non-elective contribution structure. Re-read these when the plan changes, not every cycle.

3. **Sync the census.** Call `companies_participants_list`
   (`GET /companies/{company_id}/participants/`). The response embeds each participant's current
   deductions — deferrals and loan payments — so this single call is usually enough to build the
   whole payroll. Read `todays_payment` on active loans for the per-payroll loan deduction, and
   `met_eligibility_date` to know who is contributing yet.

4. **Confirm per-participant deferrals where you need detail.** For any participant whose elections
   you want to re-read individually, call `participants_deferrals_retrieve`
   (`GET /participants/{participant_id}/deferrals/`). Respect these fields:
   - `is_percent` — `amount` is a percentage when true, a flat dollar amount when false.
   - `is_eligible` — do not deduct for an ineligible participant.
   - `hit_max` — stop deducting; the participant has reached the annual limit.
   - `active_traditional` / `active_roth` — which buckets the deduction splits into.

5. **Calculate the lines.** For each participant build a payroll line:
   - `participant_id` (required) — the encrypted participant identifier.
   - `hours` (required) — hours worked this period.
   - `gross_pay` (required) — gross pay **eligible for 401(k) deductions only**. Exclude
     reimbursements and any bonus that does not count toward 401(k) compensation.
   - `pre_tax_amount` — required if the participant has pre-tax deferrals.
   - `post_tax_amount` — required if the participant has post-tax (Roth) deferrals.
   - `company_contribution` — the employer match amount, optional.
   - `other_additions[]` — loan deductions, each with `additional_amount` and an `other_type` of
     exactly `"Loan Principal"` or `"Loan Interest"`.
   - `hours_ytd` / `gross_pay_ytd` — write-only; supplying them updates `hours_worked_ytd` and
     `compensation_ytd` on the participant record, so you get census maintenance for free.

6. **Submit the file — idempotently.** Call `companies_submit_payroll_create`
   (`POST /companies/{company_id}/submit-payroll/`) with `check_date` (required, the W2 pay date),
   optional `pay_period_start` / `pay_period_end` / `is_off_cycle`, and `payroll_lines`.

   **Always send an `Idempotent-Key` header.** This is one of only two operations on the whole API
   that accepts one, and it is the one that moves money. 401GO remembers the key for **24 hours**,
   so a retry inside that window will not double-submit contributions. Generate one key per logical
   payroll (e.g. a UUID derived from company + check_date) and reuse the same key on every retry of
   that payroll — never a fresh key per attempt.

   The response returns the created payroll file including the calculated `ach_date` and derived
   `pre_tax_percent` / `post_tax_percent`. Store the file's id for reconciliation.

7. **Reconcile.** Call `participants_payroll_lines_list`
   (`GET /participants/{participant_id}/payroll-lines/`) to confirm what 401GO recorded against a
   participant, and `participants_totals_retrieve` for year-to-date totals and remaining limits.

## Error handling

Errors are **not** RFC 9457. Every error is `{"user_error_message": ..., "developer_error_detail": ...}`,
served as `application/json`. Show `user_error_message` to a human; log `developer_error_detail`.
Validation failures are keyed by the offending field and nest to mirror your submitted JSON — read
the leaf pairs to find the exact field.

- **400** — a payroll line failed validation. The response keys tell you which line field.
- **401** — token missing or expired (60-minute lifetime). Refresh and retry.
- **403** — either the token lacks `company:write`, or this endpoint+method is not on your allow list.
  These are different fixes; check your granted access list before assuming a scope problem.
- **No 429 is declared** and no rate-limit policy is published. Back off conservatively anyway.

## Notes

- Contribution limits are shared: pre-tax and post-tax draw on one combined limit, so $10,000 pre-tax
  against a $23,000 limit leaves $13,000 across **both** buckets. `participants_totals_retrieve`
  returns `year_contribution_limit`, the absolute ceiling including catch-up.
- Catch-up eligibility is asymmetric — a participant can be eligible for Roth catch-up without being
  eligible for traditional catch-up. Do not infer one from the other.
