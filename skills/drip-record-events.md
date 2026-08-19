---
generated: '2026-08-13'
method: generated
name: Record subscriber events in Drip
description: Send custom behavioral events to Drip one at a time or in batches, and discover which event actions an account already uses so automations keep matching.
api: openapi/drip-events-api-openapi.yml
operations: [listEventActions, recordEvent, batchEvents]
source: >-
  operationIds verified verbatim in openapi/drip-events-api-openapi.yml.
  Batch sizing and limits from https://developer.drip.com/#rate-limiting,
  captured in rate-limits/drip-rate-limits.yml.
---

# Record subscriber events in Drip

Custom events are how behaviour from your product reaches Drip's automations. An event names an action a subscriber took; workflows and rules trigger on the action string, which is why step 1 is not optional.

## Auth
- API token via HTTP Basic, or an OAuth bearer token with the `write` scope.
- `Content-Type: application/json`; parameters go in the body, never the query string.

## Idempotency
- **This is the flow where Drip's lack of idempotency keys bites.** Unlike subscribers and orders, an event has no natural key — `recordEvent` appends. A retried request records the event twice, and any workflow triggering on that action fires twice.
- Mitigation: treat a timeout as ambiguous rather than failed, and carry your own dedupe (a processed-event-id set on your side) before re-sending. See `conventions/drip-conventions.yml`.

## Steps
1. **List the actions already in use** — `listEventActions` (`GET /v2/{account_id}/event_actions`) returns the custom event action names on the account. Reuse an existing string exactly. A near-miss (`Completed Order` vs `completed order`) creates a new action that no existing workflow listens to, and the failure is silent — the API returns success.
2. **Record a single event** — `recordEvent` (`POST /v2/{account_id}/events`) with the subscriber's `email`, the `action` string, and an optional `properties` object of your own fields. Counts against the standard 3,600/hour budget.
3. **Or record in bulk** — `batchEvents` (`POST /v2/{account_id}/events/batches`) with `{"batches": [{"events": [ ... ]}]}`, up to **1,000 events per request** and **50 requests per hour**. Drip's own guidance is to use this "when you need to record a collection of events at once that will likely exceed the regular rate limit of 3,600 requests per hour."
4. **Confirm downstream** — the `subscriber.performed_custom_event` webhook fires for recorded events. See `asyncapi/drip-webhooks.yml`.

## Errors
- `422` `presence_error` — a required attribute (usually `email` or `action`) is missing; `attribute` names it.
- `422` `email_error` — the subscriber address is malformed.
- `422` `format_error` — a resource identifier or object is not formatted correctly.
- `429` — batch budget exhausted (50/hour) or standard budget exhausted (3,600/hour). The global limiter returns the flat `{"message","documentation"}` shape with no `Retry-After`.
- See `errors/drip-problem-types.yml`.

## Choosing single vs batch
| | `recordEvent` | `batchEvents` |
|---|---|---|
| Budget | 3,600/hour | 50/hour |
| Records per request | 1 | up to 1,000 |
| Effective ceiling | 3,600 events/hour | 50,000 events/hour |
| Latency | immediate | queued in a batch |

Use `recordEvent` for interactive, low-volume signals where timing matters. Use `batchEvents` for anything replayed, backfilled, or high-volume.

## Notes
- Event properties are also the payload Custom Dynamic Content can read at email render time, inside a Workflow, as `event[...]` query parameters. See `components/drip-components.yml`.
