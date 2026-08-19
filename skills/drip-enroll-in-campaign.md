---
generated: '2026-08-13'
method: generated
name: Enroll a subscriber in an email series campaign
description: Find an active Drip Email Series Campaign, subscribe someone to it, and verify the enrollment — including the account-state transition errors that block activation.
api: openapi/drip-campaigns-api-openapi.yml
operations: [listCampaigns, getCampaign, activateCampaign, pauseCampaign, subscribeToCampaign, listCampaignSubscribers]
source: >-
  operationIds verified verbatim in openapi/drip-campaigns-api-openapi.yml.
  Transition-error and state semantics from
  https://developer.drip.com/#transition-errors, captured in
  errors/drip-problem-types.yml.
---

# Enroll a subscriber in an email series campaign

An Email Series Campaign is Drip's multi-email sequence. This is the flow an agent runs when a person qualifies for a nurture track.

## Auth
- API token via HTTP Basic, or an OAuth bearer token. `subscribeToCampaign`, `activateCampaign` and `pauseCampaign` are mutations and need the `write` scope.
- See `authentication/drip-authentication.yml` and `scopes/drip-scopes.yml`.

## Steps
1. **List campaigns** — `listCampaigns` (`GET /v2/{account_id}/campaigns`). Paginated at 100 per page via the `page` parameter; read `meta.total_pages` before assuming you have them all.
2. **Inspect the target** — `getCampaign` (`GET /v2/{account_id}/campaigns/{campaign_id}`). Check `status`, `double_optin`, `start_immediately`, `localize_sending_time` and `days_of_the_week_mask` before enrolling anyone — those determine when the first email actually lands.
3. **Activate if paused** — `activateCampaign` (`POST /v2/{account_id}/campaigns/{campaign_id}/activate`). A paused campaign accepts subscribers but does not send.
4. **Subscribe the person** — `subscribeToCampaign` (`POST /v2/{account_id}/campaigns/{campaign_id}/subscribers`). The subscriber is identified by email; if they do not exist yet they are created, so this doubles as an upsert.
5. **Verify** — `listCampaignSubscribers` (`GET /v2/{account_id}/campaigns/{campaign_id}/subscribers`) to confirm the enrollment, or listen for the `subscriber.subscribed_to_campaign` webhook.
6. **Pause when done** — `pauseCampaign` (`POST /v2/{account_id}/campaigns/{campaign_id}/pause`) stops sending without removing enrolled subscribers.

## Errors
- `403` with a **transition error** — the state change is not allowed. The most common is `no_postal_address_error`: Drip will not activate a sending campaign until the account has a default postal address, because CAN-SPAM requires one. Transition errors may carry a `code` with no `message`, so do not depend on `message` being present.
- `404` `not_found_error` — wrong `account_id` or `campaign_id`. Check the path segment, not just the id.
- `403` `authorization_error` — the token is valid but scoped to a different account, or holds only `public` where `write` is needed.
- See `errors/drip-problem-types.yml`.

## Deprecated fields
Do not write `send_to_confirmation_page`, `use_custom_confirmation_page` or `post_confirmation_url` on a campaign. All three are marked Deprecated in the reference with no announced removal date. See `lifecycle/drip-lifecycle.yml`.

## Related events
`subscriber.subscribed_to_campaign`, `subscriber.removed_from_campaign`, `subscriber.unsubscribed_from_campaign` and `subscriber.completed_campaign` all fire on this flow. See `asyncapi/drip-webhooks.yml`.
