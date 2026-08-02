---
name: Handle 401GO participant money movement — loans, disbursements and rollovers
description: Check what a participant is eligible to take out before asking, create and sign a loan request, create a split pre-tax/post-tax disbursement, record a rollover, and reconcile everything against money-movement history.
api: openapi/401go-openapi-original.json
base_url: https://app.401go.com/api
operations:
  - participants_loan_requests_info_retrieve
  - participants_loan_requests_create
  - participants_loan_requests_signature_update
  - participants_loan_requests_list
  - participants_loan_requests_retrieve
  - participants_loan_requests_update
  - participants_loan_requests_destroy
  - participants_disbursements_info_retrieve
  - participants_disbursements_create
  - participants_disbursements_list
  - participants_disbursements_retrieve
  - participants_disbursements_update
  - participants_disbursements_destroy
  - participants_rollovers_create
  - participants_rollovers_list
  - participants_rollovers_retrieve
  - participants_rollovers_update
  - participants_rollovers_destroy
  - participants_money_movement_history_list
  - participants_totals_retrieve
scopes:
  - participant:read
  - participant:write
generated: '2026-08-02'
method: generated
source: https://developer.401go.com/reference/participants_disbursements_create
---

# Handle 401GO participant money movement

Nineteen of 401GO's 72 operations are money movement. The discipline that matters: **always call the
`-info` endpoint first**. It tells you what the participant is actually eligible for and what the
available amounts are, so you never present an option that will be rejected.

## Loans

### 1. Check eligibility and limits

`participants_loan_requests_info_retrieve` (`GET /participants/{participant_id}/loan-requests-info/`)
returns the information you need before beginning a loan request. Also confirm
`loans_permitted` on the plan via `companies_plan_provisions_retrieve` — plans can disallow loans
entirely.

### 2. Create the request

`participants_loan_requests_create` (`POST /participants/{participant_id}/loan-requests/`).

Watch the duration rule: a loan may not exceed **5 years** unless `is_primary_residence` is true.
Violating it returns a 400 keyed on `loan_duration` with
`developer_error_detail: "cannot be more than 5 years if is_primary_residence is False"`.

### 3. Sign it

`participants_loan_requests_signature_update`
(`PUT /participants/{participant_id}/loan-requests-signature/{loan_request_id}/`) submits the
signature. **A loan request is not complete until signed** — treat create + sign as one flow and
surface the unsigned state clearly if the participant abandons it.

### 4. Manage

`participants_loan_requests_list`, `participants_loan_requests_retrieve`,
`participants_loan_requests_update` (PUT, full replacement) and `participants_loan_requests_destroy`
(DELETE, 204 no content).

### 5. Feed the deduction back into payroll

An active loan appears on the participant record with `todays_payment` — the amount due per payroll.
Your payroll integration must add it as an `other_additions[]` entry on the payroll line, with
`other_type` of exactly `"Loan Principal"` or `"Loan Interest"`. See the *Run a 401GO payroll
contribution cycle* skill.

## Disbursements

### 1. Get available amounts for the right disbursement type

`participants_disbursements_info_retrieve`
(`GET /participants/{participant_id}/disbursements-info/`) accepts query flags that change the
computed amounts. All default to false:

- `is_hardship` — hardship disbursement amounts
- `is_emergency` — emergency expense amounts
- `is_move_rollover` — move rollover amounts

Call it with the flag matching the disbursement the participant is asking for, not bare.

### 2. Create, minding the split-field pattern

`participants_disbursements_create` (`POST /participants/{participant_id}/disbursements/`).

**Single disbursement (pre-tax only, or post-tax only):**
- Use the regular fields (`payment_method`, `bank_name`, …) for payment details.
- `pretax_memo` for pre-tax; `posttax_memo` for post-tax.
- `payment_address` for pre-tax; `posttax_payment_address` for post-tax.

**Split disbursement (both pre-tax and post-tax):**
- Regular fields carry the **pre-tax** portion.
- `posttax_*` fields carry the post-tax portion when it differs from pre-tax.
- **Both `pretax_memo` and `posttax_memo` are required.**
- Both `payment_address` and `posttax_payment_address` may be specified if they differ.

The trap is that the "regular" fields are not neutral — in a split they mean *pre-tax*.

### 3. Manage

`participants_disbursements_list`, `participants_disbursements_retrieve`,
`participants_disbursements_update` (PUT) and `participants_disbursements_destroy` (DELETE, 204).

## Rollovers

`participants_rollovers_create` (`POST /participants/{participant_id}/rollovers/`) creates a rollover
**in one single request** — there is no multi-step draft. Manage with
`participants_rollovers_list`, `participants_rollovers_retrieve`, `participants_rollovers_update`
and `participants_rollovers_destroy`.

## Reconcile

`participants_money_movement_history_list`
(`GET /participants/{participant_id}/money-movement-history/`) is the ledger. Read `account_type`:

- `Pre-tax` — employee pre-tax movements
- `Post-tax` — employee post-tax movements
- `Vested Contribution` — vested employer contributions
- `Non-vested Contribution` — non-vested employer contributions

`tax_type` is only meaningful for distinguishing pre-tax from post-tax **employer** contributions,
and only when it carries a value. Do not branch on it elsewhere.

`participants_totals_retrieve` (`GET /participants/{participant_id}/totals/`) gives balances,
active loan balances, cash balance and the annual limits.

## Error handling

Custom envelope, not RFC 9457: `{"user_error_message", "developer_error_detail"}`, nesting to mirror
your submitted JSON.

- **400** — eligibility or field validation. Almost always avoidable by calling the `-info` endpoint
  first.
- **401** — expired 60-minute access token.
- **403** — missing `participant:write`, or the endpoint+method is not on your allow list.
- **404** — declared on the document-retrieval operations; the request or resource does not exist
  for this participant.

**No money-movement operation accepts an `Idempotent-Key`** — only `companies_submit_payroll_create`
and `participants_beneficiaries_create` do. These are irreversible, money-moving creates with no
replay protection. Never blind-retry a POST whose response you did not observe: re-read the
corresponding list endpoint and match on your own reference data before retrying.
