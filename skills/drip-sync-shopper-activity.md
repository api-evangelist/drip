---
generated: '2026-08-13'
method: generated
name: Sync ecommerce shopper activity to Drip
description: Push product catalog, cart and order activity into Drip's v3 Shopper Activity API in the right order, and avoid the deprecated v2 Orders surface.
api: openapi/drip-shopper-activity-api-openapi.yml
operations: [batchShopperProducts, batchShopperCarts, batchShopperOrders, createOrUpdateOrder]
source: >-
  operationIds verified verbatim in
  openapi/drip-shopper-activity-api-openapi.yml and
  openapi/drip-orders-api-openapi.yml. Legacy/replacement relationship from
  https://developer.drip.com/ ("Orders (Legacy)") captured in
  lifecycle/drip-lifecycle.yml.
---

# Sync ecommerce shopper activity to Drip

Shopper Activity is Drip's ecommerce ingestion surface and the reason Drip exists as an ecommerce-first platform. It lives on **v3**, a different major from the rest of the API.

## Which surface to use
Drip has two order paths and they are not equivalent:

- **Use** `batchShopperOrders` — `POST /v3/{account_id}/shopper_activity/order/batch`.
- **Avoid for new work** `createOrUpdateOrder` — `POST /v2/{account_id}/orders`. The reference files this under **"Orders (Legacy)"**. It still works and Drip has announced no removal date, but it is superseded. Note that this repo's `openapi/drip-orders-api-openapi.yml` describes the legacy surface. See `lifecycle/drip-lifecycle.yml`.

## Auth
- API token via HTTP Basic, or an OAuth bearer token with the `write` scope.
- Base host is the same (`https://api.getdrip.com`); only the version segment changes to `/v3`.

## Idempotency
- No idempotency key header. These writes are create-or-update on the identifiers you supply (provider order id, cart id, product id), so replaying the same payload converges rather than duplicating — which is exactly what makes a nightly re-sync safe. See `conventions/drip-conventions.yml`.

## Steps
1. **Products first** — `batchShopperProducts` (`POST /v3/{account_id}/shopper_activity/product/batch`). Carts and orders reference products; loading the catalog first means line items resolve rather than dangle.
2. **Carts** — `batchShopperCarts` (`POST /v3/{account_id}/shopper_activity/cart/batch`). This is what powers abandoned-cart automations, so it is usually the highest-frequency of the three.
3. **Orders** — `batchShopperOrders` (`POST /v3/{account_id}/shopper_activity/order/batch`).
4. **Backfill vs steady state** — for an initial catalog load, run products to completion before opening the cart/order taps. Steady state, interleave them but keep the ordering per entity (a cart referencing a product you have not sent yet has nothing to resolve to).

## Rate limits
- All three are batch endpoints: **50 requests/hour**, **1,000 records each** — a combined ceiling of 50,000 records/hour across the three, since they share the batch budget with subscribers, unsubscribes and events.
- Plan the split explicitly. A store syncing carts every minute will exhaust the batch budget on carts alone and starve the order sync.
- See `rate-limits/drip-rate-limits.yml`.

## Errors
- `422` `format_error` — an identifier or nested object is malformed.
- `422` `range_error` — a numeric value is out of range. Note Drip caps conversion values at 2,147,483,647; treat monetary fields as bounded 32-bit integers in the smallest currency unit.
- `422` `presence_error` — a required attribute is missing; `attribute` names it.
- `429` — batch budget exhausted. No `Retry-After` on the global limiter; back off for the hour.
- See `errors/drip-problem-types.yml`.

## Related events
Shopper activity drives `subscriber.updated_lifetime_value` and feeds the lead-scoring events (`subscriber.updated_lead_score`, `subscriber.became_lead`). See `asyncapi/drip-webhooks.yml`.
