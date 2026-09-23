---
name: scanverity-assess-market
description: >-
  Request a resolution-risk assessment for one Polymarket market from the Scanverity Resolution API
  and poll it to a terminal state, handling the withheld and failed outcomes correctly. Use when
  asked how likely a prediction market is to resolve ambiguously, badly, or against its title.
api: Scanverity Resolution API
base_url: https://scanverity.com
generated: '2026-09-04'
method: generated
source: openapi/scanverity-resolution-api-openapi.json
operations:
  - createResolutionAssessment
  - getResolutionAssessment
---

# Assess one Polymarket market

Scanverity assesses **resolution risk** — the chance a market settles in a way its title does not
lead you to expect. It does not predict outcomes and gives no trading advice.

## Before you start

- You need a Bearer token with the `resolution:request` and `resolution:read` scopes.
- Access is **private beta and feature-gated**. A syntactically valid token still needs an entitled,
  allowlisted account; the published contract explicitly does not assert the production flag is on.
- Develop against `svr_sandbox_` first. Sandbox calls are free, deterministic, and never touch the
  live resolver.

## Step 1 — create the assessment

`createResolutionAssessment` — `POST /v1/resolution-assessments`

- **`Idempotency-Key` is required, not optional.** Use 1–255 printable ASCII characters. Derive it
  from the market plus your own request identity so a retry replays instead of re-charging.
- Body: `market` (required — a Polymarket market id, supported slug, or supported canonical URL),
  optional `customer_reference` (≤200 chars, echoed verbatim, never interpreted), optional
  `webhook: true`.
- Returns **202** with `status: accepted`, an `assessment_id` matching `^ra_[a-f0-9]{32}$`, and a
  `polling_guidance` block.

Replay behaviour: the same key with the **same** canonical payload returns the existing resource
with 202 and an `Idempotent-Replayed: true` header — never a second assessment, never a second
charge. The same key with a **different** payload returns **409 `IDEMPOTENCY_CONFLICT`**. The
binding is scoped to account plus token and lasts exactly 30 days.

Do not set `webhook: true` unless you have already registered an active endpoint in the same account
*and* the same token environment — otherwise you get **422 `WEBHOOK_NOT_CONFIGURED`**. Inline
webhook URLs are never accepted.

## Step 2 — poll to a terminal state

`getResolutionAssessment` — `GET /v1/resolution-assessments/{assessment_id}`

Poll at `polling_guidance.recommended_interval_seconds` (published example: 3s). Typical completion
is ~10s, published as indicative behaviour and explicitly **not** a guarantee.

**Reads and polls are never billable**, so poll rather than guess. Stay inside the read bucket:
120 req/min live, 30 req/min sandbox.

## Step 3 — branch on the terminal status

`status` is the discriminator. There are three terminal states and only one of them costs money.

| `status` | Meaning | Billable |
|---|---|---|
| `released` | A risk figure was produced. Read `resolution_risk`, `risk_factors`, `provenance`. | only if `billable: true` |
| `withheld` | Scanverity **declined to state a figure**. Read `withheld.code` and `withheld.reason`. | never |
| `failed` | The run terminated in error. Read `error.code` and `error.retryable`. | never |

**Do not treat `withheld` as an error or as "low risk".** It is a successful call in which the
provider is telling you it could not verify enough to answer:

- `WITHHELD_NO_RULES_TEXT` — the rule text could not be retrieved or verified.
- `WITHHELD_NO_FINITE_ESTIMATE` — no finite calibrated estimate could be produced.
- `WITHHELD_SOURCE_UNVERIFIED` — the resolution source could not be verified.

The withheld body carries `could_not_verify`, `reason`, any `verified_facts` established anyway, and
a `retry.policy`. Surface that reasoning; never substitute a number of your own.

On `failed`, retry only when `error.retryable` is true.

## Billing rules worth knowing before you loop

A call bills only when `status` is `released` **and** `billable` is true **and**
`metering.disposition` is `billable`. Never billable: idempotent duplicates, cache hits, reads,
polls, any 4xx, any 5xx, timeouts, failed assessments, withheld assessments, webhook traffic, and
every sandbox call.

## Errors

`INVALID_MARKET` (400) — unparseable identifier. `UNSUPPORTED_MARKET` (422) — outside coverage.
`INSUFFICIENT_SCOPE` (403) — the `detail` field names the missing scope. `INVALID_TOKEN` (401) —
unknown, malformed, expired and revoked all share this one response by design. `RATE_LIMITED` (429)
— honour `Retry-After`. Bodies are `{ "error": { "code", "message", "detail"?, "docs_url"? } }`.

Never put the token in the URL; query-string tokens are rejected.
