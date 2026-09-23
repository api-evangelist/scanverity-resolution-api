---
name: scanverity-manage-webhooks
description: >-
  Register, verify, audit and remove signed webhook endpoints on the Scanverity Resolution API,
  including HMAC signature verification and audited manual redelivery. Use when wiring push
  delivery of terminal assessment events instead of polling.
api: Scanverity Resolution API
base_url: https://scanverity.com
generated: '2026-09-04'
method: generated
source: openapi/scanverity-resolution-api-openapi.json
operations:
  - createResolutionWebhookEndpoint
  - listResolutionWebhookEndpoints
  - deleteResolutionWebhookEndpoint
  - listResolutionWebhookDeliveries
  - redeliverResolutionWebhook
---

# Signed webhooks

Every operation here needs the `webhooks:manage` scope. Endpoints are scoped to **account and
environment** — a `svr_sandbox_` token cannot see or touch a live endpoint, and cross-environment
identifiers return `NOT_FOUND`, deliberately indistinguishable from unknown ids.

## Register an endpoint

`createResolutionWebhookEndpoint` — `POST /v1/webhook-endpoints` → **201**

The destination must be credential-free public HTTPS on port 443 and must pass DNS/SSRF validation;
anything else returns `UNSAFE_WEBHOOK_TARGET`. You may retain at most **ten** endpoint records per
environment — the eleventh returns `WEBHOOK_ENDPOINT_LIMIT`. The registration body is capped at
16 KiB (413 beyond).

Subscribe to one to three events: `assessment.released`, `assessment.withheld`, `assessment.failed`.

**The `signing_secret` (prefix `svrwhsec_`) is revealed exactly once, in this response.** It is
encrypted at rest and never returned by any later read. Store it before you do anything else; if you
lose it your only recovery is to delete the endpoint and register a new one.

## Verify every delivery

Header: `Scanverity-Signature`, formatted `t=<unix>, v1=<hex>`.

`v1` is `HMAC-SHA256(signing_secret, "<t>.<raw request body>")`. Verify against the **raw** body
before any JSON parsing, compare in constant time, and reject anything where `|now - t|` exceeds the
**300-second** tolerance. Reject unverified payloads outright — do not act on them.

`Scanverity-Delivery-Id` is stable across automatic attempts, so use it to deduplicate. A manual
redelivery arrives with a **new** id linked by `redelivery_of`.

## Delivery guarantees

Retries run at **60s, 300s, 1800s, 7200s, 28800s** — at most **5 attempts per delivery group**.
Three days of continuous failure auto-disables the endpoint. All delivery, retry and redelivery
traffic is non-billable.

## Audit what happened

`listResolutionWebhookDeliveries` — `GET /v1/webhook-endpoints/{endpoint_id}/deliveries`

Returns the newest 50 delivery groups, newest first, with append-only attempt starts and outcomes.
Evidence is retained **90 days**. States: `pending`, `delivered`, `failed`, `cancelled`.

## Redeliver

`redeliverResolutionWebhook` — `POST /v1/webhook-endpoints/{endpoint_id}/redeliver/{delivery_id}`
→ **202**

Works only on an **enabled** endpoint and a **terminal** source group. It creates a distinct
non-billable delivery group with a new `delivery_id` and `redelivery_of` pointing at the original.
The original group is never rewritten. A still-pending source group returns `NOT_FOUND`.

## Remove

`deleteResolutionWebhookEndpoint` — `DELETE /v1/webhook-endpoints/{endpoint_id}` → **204**

This is **destructive and not reversible**: the destination and signing secret are erased and
pending work is cancelled. Bounded append-only delivery evidence is preserved regardless. Treat this
as a human-approval step.
