---
name: scanverity-sandbox-fixtures
description: >-
  Exercise every terminal outcome of the Scanverity Resolution API against its deterministic sandbox
  fixture catalog, free and without an entitled live account. Use when building or testing an
  integration before production access is granted.
api: Scanverity Resolution API
base_url: https://scanverity.com
generated: '2026-09-04'
method: generated
source: https://scanverity.com/resolution-api/docs
operations:
  - createResolutionAssessment
  - getResolutionAssessment
---

# Build against the deterministic sandbox

The sandbox is selected by **token prefix**, not by a request flag. Authenticate with an
`svr_sandbox_` token and the identical request shape resolves against the closed fixture catalog
`resolution-sandbox-v1` instead of the live resolver.

Guarantees the provider publishes: it never calls the live resolver, never reads provider or
customer data, is always non-billable, and is reproducible. Released fixtures carry a fixed
`assessed_at` of `2026-08-03T00:00:00.000Z`.

## The fixtures

Pass one of these as the `market` value on `createResolutionAssessment`:

| `market` | Terminal outcome | What it exercises |
|---|---|---|
| `sv-sandbox-released-calibrated` | `released` | A full calibrated risk figure |
| `sv-sandbox-released-modelled-only` | `released` | Modelled risk with no calibration |
| `sv-sandbox-withheld-no-rules` | `withheld` | `WITHHELD_NO_RULES_TEXT` |
| `sv-sandbox-withheld-no-finite-estimate` | `withheld` | `WITHHELD_NO_FINITE_ESTIMATE` |
| `sv-sandbox-withheld-source-unverified` | `withheld` | `WITHHELD_SOURCE_UNVERIFIED` |
| `sv-sandbox-failed` | `failed` | A readable terminal failed resource |
| `sv-sandbox-rate-limit` | `released` | Rate-limit behaviour |
| `sv-sandbox-ambiguous` | error | `INVALID_MARKET` |
| `sv-sandbox-unsupported` | error | `UNSUPPORTED_MARKET` |

Cover all three terminal states plus both rejection paths before you ship — the `withheld` branch is
the one integrations most often get wrong, because it is a **success** that deliberately declines to
give a number.

## Differences from live

- Sandbox rate limits are much tighter: **30 reads/min and 10 requests/min**, versus 120 and 60 live.
- Sandbox does **not** use the request-level `PROVIDER_UNAVAILABLE` fallback. Use the
  `sv-sandbox-failed` fixture to exercise terminal failure instead.
- Webhook endpoints are environment-scoped, so register a sandbox endpoint to test delivery; a
  sandbox token cannot reach a live endpoint.
- `Idempotency-Key` is still required, and replay semantics behave identically.

## Then move to live

The only changes are the token prefix (`svr_live_`), real market identifiers, higher rate limits,
and the fact that a `released` assessment with `billable: true` now costs money. Everything else —
request shape, polling, statuses, error codes — is identical.
