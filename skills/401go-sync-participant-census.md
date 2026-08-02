---
name: Sync employee census into 401GO
description: Keep a payroll or HCM system's employee roster in sync with 401GO — create and update participants, handle the SSN-based upsert and the email-change side effect, and read back setup state and eligibility events.
api: openapi/401go-openapi-original.json
base_url: https://app.401go.com/api
operations:
  - companies_participants_list
  - companies_participants_create
  - companies_participants_retrieve
  - companies_participants_update
  - companies_participants_partial_update
  - participants_participant_setup_retrieve
  - participants_events_retrieve
  - participants_events_create
scopes:
  - company:read
  - participant:read
  - participant:write
generated: '2026-08-02'
method: generated
source: https://developer.401go.com/docs/payroll-integration
---

# Sync employee census into 401GO

Census sync is the first thing a payroll or HCM integration does and the thing it keeps doing.
401GO's participant endpoints have two behaviours that will surprise you if you treat them as plain
CRUD: **create is an SSN-keyed upsert**, and **changing an email creates a second login**.

## Reading the roster

`companies_participants_list` (`GET /companies/{company_id}/participants/`) returns the roster and
embeds each participant's current deductions — deferrals and loan payments — so one call gives you
both the census and the deduction instructions. Paginate with `page` / `page_size`; the envelope is
`count` / `next` / `previous` / `results`.

For a single record use `companies_participants_retrieve`
(`GET /companies/{company_id}/participants/{participant_id}/`), which likewise includes deferrals
and loan payments.

## Creating participants

`companies_participants_create` (`POST /companies/{company_id}/participants/`) accepts a single
participant **or an array** of them.

Required:
- `ssn` — 9 digits
- `start_date` — hire date
- `name` — full name
- `email` **or** `phone` — at least one contact method

Optional: `dob`, `termination_date`, `hours_worked_ytd`, `compensation_ytd`,
`years_worked_1000_hours`, `prior_year_total_compensation`, `ownership_percentage`,
`company_officer`, and a nested `address`.

If `address` is supplied, `address_line_1`, `city`, `state` and `postal_code` are all required;
`address_line_2` is optional.

**You cannot set `deductions` or `met_eligibility_date` on create.** Those are 401GO-owned.

**Create is an upsert.** Posting a participant whose `ssn` already exists on the plan updates that
employee rather than erroring. This makes a full-roster replay safe, but it also means a typo'd SSN
silently creates a new person. Validate SSNs before sending.

**Send an `Idempotent-Key` header** on create. This is one of only two operations on the API that
honours it (the other is payroll submission); 401GO remembers the key for 24 hours. Note: the header
is spelled `Idempotent-Key`, not `Idempotency-Key`.

## Updating participants

- `companies_participants_update` — `PUT /companies/{company_id}/participants/{participant_id}/`
- `companies_participants_partial_update` — `PATCH` on the same path

Updates require `object_id` on the record. You still cannot add deductions or change
`met_eligibility_date`.

**The email-change side effect:** changing a participant's email does **not** move their login. It
creates an *additional* login with the new email so the employee is not locked out of the old one.
Do not treat email as a mutable identity key, and do not "fix" duplicate logins by re-sending
emails. Read-only fields you receive back include `met_eligibility_date`, `deferrals`, `loans`,
`timestamp` and `timestamp_updated`.

## Cheaper census maintenance via payroll

`hours_ytd` and `gross_pay_ytd` on a payroll line are write-only fields that update
`hours_worked_ytd` and `compensation_ytd` on the participant record. If you submit payroll through
401GO you get YTD maintenance without a separate participant update. See the
*Run a 401GO payroll contribution cycle* skill.

## Checking setup and eligibility state

- `participants_participant_setup_retrieve`
  (`GET /participants/{participant_id}/participant-setup/`) returns the participant's plan data and
  whether they have set deferral elections. `plan_type` is one of
  `Volume Submitter Prototype Format` (401k plans), `403B`, or `Starter-K`.
- `participants_events_retrieve` (`GET /participants/{participant_id}/events/`) lists the event
  types recorded for a participant. **To decide whether a participant's dashboard is reachable,
  check for the `completed setup` event** — that is the documented signal.
- `participants_events_create` (`POST /participants/{participant_id}/events/`) adds event types. It
  will not duplicate an event that already exists, so it is safe to replay.

## Ongoing loop

1. Detect hires, terminations and demographic changes in your system.
2. `POST` new hires (upsert-safe), `PATCH` changes, set `termination_date` on leavers.
3. Re-read the roster with `companies_participants_list` before each payroll to pick up
   401GO-side changes — new eligibility, changed deferrals, new loan payments.

## Error handling

Errors use 401GO's custom envelope, not RFC 9457:
`{"user_error_message": ..., "developer_error_detail": ...}`, nested to mirror your submitted JSON
(so a bad `address.city` comes back keyed under `address` → `city`).

- **400** — validation. Read the field-keyed leaves.
- **401** — expired token (60-minute access tokens).
- **403** — missing `participant:write`, *or* this endpoint+method is not on your client's allow
  list. Check <https://developer.401go.com/docs/api-endpoint-and-method-access>.

Handle PII carefully: these payloads carry SSNs, dates of birth and home addresses. Do not log
request bodies.
