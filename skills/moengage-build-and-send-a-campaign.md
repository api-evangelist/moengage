---
name: Build, validate, test and publish a MoEngage campaign
description: Drive the MoEngage Campaigns V5 draft lifecycle — create a draft, patch components, validate, test-send, then change status — using the Idempotency-Key contract correctly.
api: openapi/moengage-campaign-draft-openapi.yml
operations:
  - create_draft_campaign_v5
  - patch_draft_campaign_v5
  - validate_draft_campaign_v5
  - preview_personalized_content_v5
  - test_campaign_v5
  - change_campaign_status_v5
  - get_single_campaign_v5
  - search_campaigns_v5
  - get_campaign_meta_v5
---

# Build, validate, test and publish a MoEngage campaign

The Campaigns V5 API builds a campaign **incrementally** rather than in one payload. Do not try to
assemble a complete campaign in a single call.

## 0. Idempotency — read this first

Every `POST` and `PATCH` on this API requires an `Idempotency-Key` header containing a **UUID v4**.

- Reuse the **same** key only when retrying an identical request after a network failure or timeout.
- Use a **new** key for every logically distinct operation and for any retry whose body has changed.
- `400 MISSING_IDEMPOTENCY_KEY` / `400 INVALID_IDEMPOTENCY_KEY` — the header is absent or not a UUID v4.
- `409 DUPLICATE_IDEMPOTENCY_KEY` — a request with that key is in flight; wait, do not spin.
- `409 IDEMPOTENCY_CONFLICT` — the key was reused with a different body; generate a new key.

`validate_draft_campaign_v5` is the documented exception and does **not** require the header.

## 1. Create the draft

`create_draft_campaign_v5` returns a draft campaign. The campaign is in `DRAFT` state and is not sending.

## 2. Patch components

`patch_draft_campaign_v5` updates one or more components — content, audience/segment, trigger conditions,
scheduling. Patch repeatedly; each call is a separate idempotent operation with its own key.

`409` from this operation means the campaign is no longer in `DRAFT` (it is `ACTIVE`, `PAUSED` or
`STOPPED`). Only drafts are patchable — read state first with `get_single_campaign_v5`.

## 3. Preview and validate

- `preview_personalized_content_v5` renders personalization for sample users. Use it to catch Jinja and
  merge-field mistakes before anything is sent.
- `validate_draft_campaign_v5` dry-runs the full publish validation and returns field-level errors
  without changing the draft. Always run it before test-sending.

## 4. Test send

`test_campaign_v5` delivers a test message so a human can confirm rendering on a real device. There is no
sandbox host for campaigns — test sends go to the production host against real recipients you nominate.

## 5. Change status

`change_campaign_status_v5` performs `STOP`, `PAUSE`, or `RESUME` on a **running** campaign — not on a
draft. A `422` means the action is invalid for the campaign's current state or delivery type:

- `STOP` on a Periodic campaign
- `PAUSE`/`RESUME` on a One-time campaign
- `RESUME` on a campaign that is not `PAUSED`
- `PAUSE` on a campaign that is not `ACTIVE`, `SCHEDULED`, or `SENDING`

## Finding campaigns

`search_campaigns_v5` filters by name, status, ID, channel, delivery type, and date range. Include
`DRAFT` in the status filter to see drafts. `get_campaign_meta_v5` returns metadata only, which is
cheaper than a full read. Note the published V1-vs-V5 behavioural differences for search.

## Rate limits

The Campaigns API is the tightest surface on the platform: **5 requests/min, 25/hour, 100/day**. Build
the draft in as few patches as the change set allows, and never poll it. Watch `x-ratelimit-remaining`.

## Errors

V5 uses the structured envelope `{response_id, error:{code, message, target, details[], doc_url}}`.
`details[]` appears only on `400 VALIDATION_FAILED` and lists every field violation at once — fix all of
them before resubmitting. Keep `response_id` for support. See `errors/moengage-problem-types.yml`.
