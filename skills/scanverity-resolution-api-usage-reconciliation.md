---
name: scanverity-reconcile-usage
description: >-
  Read reconciled monthly usage totals and per-assessment billing evidence from the Scanverity
  Resolution API, and page the event ledger correctly. Use to verify an invoice, audit what was
  charged, or check remaining included quota.
api: Scanverity Resolution API
base_url: https://scanverity.com
generated: '2026-09-04'
method: generated
source: openapi/scanverity-resolution-api-openapi.json
operations:
  - getResolutionUsageSummary
  - listResolutionUsageEvents
---

# Reconcile usage and billing

Both operations require a **live** token carrying `usage:read`. A sandbox token is refused even if
it carries the scope — the docs are explicit that these are live-only routes. The account is derived
solely from the bearer token; account selectors in the request are rejected.

## Monthly totals

`getResolutionUsageSummary` — `GET /v1/usage?period=YYYY-MM`

`period` is optional and defaults to the current UTC month. Returns `totals` with `released`,
`billable`, `evaluation_credit`, `measurement` and `adjusted_units`.

## The event ledger

`listResolutionUsageEvents` — `GET /v1/usage/events?period=YYYY-MM&limit=&cursor=`

`period` is **required** here. `limit` defaults to 50 and caps at 100.

Page with the opaque `cursor`: loop while `has_more` is true, passing `next_cursor` each time.
**Cursors are bound to the authenticated account and the requested period** — never construct one,
never reuse one across periods or scopes, or you get `INVALID_USAGE_CURSOR`.

## Reading a usage event

Each event carries `event_id` (a 26-character ULID), `assessment_id`, `market_id`, `released_at`,
`assessment_version`, and three quantity fields:

- `original_quantity` — always `1`.
- `adjustment_quantity` — `-1` or `0`.
- `effective_quantity` — the resulting `0` or `1`.

**This is the reversal mechanism for this API.** There is no void or refund operation; a charge is
taken back by an adjustment that credits the unit back to zero while leaving the original event
visible. `billing_disposition` is `billable`, `evaluation_credit`, or `measurement` — only
`billable` costs money.

## Failure modes are fail-closed

These routes serve `application/problem+json` and **never return partial data**. If the reconciled
view cannot be read safely or fails its integrity/parity checks you get `USAGE_UNAVAILABLE` or
`USAGE_INTEGRITY_FAILED` (503) with nothing partial attached — retry later rather than treating an
empty result as zero usage. `INVALID_USAGE_QUERY` (400) means `period` was not `YYYY-MM`.

Note the envelope: the media type is problem+json but the body is Scanverity's own
`{ "error": { "code", "message" } }` shape, not RFC 9457 members.
