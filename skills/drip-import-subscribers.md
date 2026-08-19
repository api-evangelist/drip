---
generated: '2026-08-13'
method: generated
name: Bulk import subscribers into Drip
description: Import or update a list of people in a Drip account using the batch subscribers endpoint, staying inside the batch rate limit and matching the account's existing custom fields.
api: openapi/drip-subscribers-api-openapi.yml
operations: [listAccounts, getAccount, listCustomFields, batchSubscribers]
source: >-
  operationIds verified verbatim in openapi/drip-accounts-api-openapi.yml,
  openapi/drip-custom-fields-api-openapi.yml and
  openapi/drip-subscribers-api-openapi.yml. Runtime rules from
  conventions/drip-conventions.yml, rate-limits/drip-rate-limits.yml and
  errors/drip-problem-types.yml, all read from https://developer.drip.com/.
---

# Bulk import subscribers into Drip

Load a list of people into a Drip account. This is the write path most integrations start with, and it is the one where the rate limit shape matters most — the single-subscriber endpoint would burn the 3,600/hour budget on a few thousand records, while one batch call carries a thousand.

## Auth
- HTTP Basic with the API token as the username and an empty password (`-u 'YOUR_API_KEY:'`), or `Authorization: Bearer <oauth_token>` with the `write` scope.
- Send `User-Agent: Your App Name (www.yourapp.com)` and `Content-Type: application/json`.
- See `authentication/drip-authentication.yml`.

## Idempotency
- Drip has **no** idempotency key header. Do not retry blindly expecting dedupe protection.
- What you get instead: `batchSubscribers` is create-or-update keyed on `email`. Replaying the same payload converges on the same subscriber rather than duplicating it. Design your retry around that natural key. See `conventions/drip-conventions.yml`.

## Steps
1. **Resolve the account** — `listAccounts` (`GET /v2/accounts`) returns every account the credential can reach; `getAccount` (`GET /v2/accounts/{account_id}`) confirms one. Capture `id`; nearly every later path is `/v2/{account_id}/...`.
2. **Read the existing custom fields** — `listCustomFields` (`GET /v2/{account_id}/custom_field_identifiers`) returns the custom field names already in use on the account. Map your source columns onto these before importing. Inventing a new field name here silently creates it, which is how accounts end up with `first_name`, `firstname` and `FirstName` side by side.
3. **Chunk the input** — at most **1,000 subscriber records per request**, and at most **50 batch requests per hour**. That is a ceiling of 50,000 records/hour. Size your chunks at 1,000 and pace at roughly one request every 72 seconds to stay under the hourly cap.
4. **Import each chunk** — `batchSubscribers` (`POST /v2/{account_id}/subscribers/batches`) with a body shaped `{"batches": [{"subscribers": [ ... ]}]}`. Each subscriber may carry `email`, `time_zone`, `tags[]` and a `custom_fields` object.
5. **Reconcile** — the import is asynchronous in effect; verify by reading subscribers back or by listening for the `subscriber.created` webhook (see `asyncapi/drip-webhooks.yml`).

## Errors
- `422` with `presence_error` on `email` — a record in the chunk has no address. The envelope is `{"errors":[{"code","attribute","message"}]}`, and `attribute` names the offending field.
- `422` with `email_error` — malformed address. Validate before sending; a bad record can cost you a whole batch slot.
- `422` with `time_error` — `time_zone` values must be valid IANA zones and timestamps must be ISO-8601.
- `429` — you have hit the 50/hour batch limit. The global limiter returns a flat `{"message","documentation"}` object with **no** `errors` array and **no** `Retry-After`, so back off for the remainder of the hour rather than parsing a reset.
- See `errors/drip-problem-types.yml`.

## Rate limits
- Batch endpoints: 50 requests/hour, 1,000 records each.
- Everything else: 3,600 requests/hour.
- Watch `X-RateLimit-Limit` and `X-RateLimit-Remaining` on every response. There is no reset header — see `rate-limits/drip-rate-limits.yml`.

## Notes
- The unsubscribe counterpart is `batchUnsubscribes` (`POST /v2/{account_id}/unsubscribes/batches`) and shares the same 50/hour budget. Budget both together.
- Drip's OpenAPI in this repo is an API Evangelist best-effort transcription, not a Drip-published contract. Single-subscriber create/fetch/delete operations exist in the human reference at https://developer.drip.com/ but are not in the spec.
