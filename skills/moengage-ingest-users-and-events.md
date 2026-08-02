---
name: Ingest users and events into MoEngage
description: Create or update user profiles, track behavioural events, register devices, and run a bulk import against the MoEngage Data API using workspace-scoped Basic Auth.
api: openapi/moengage-data-openapi.yml
operations:
  - POST /customer/{app_id}
  - POST /event/{Workspace_ID}
  - POST /device/{app_id}
  - POST /transition/{Workspace_ID}
  - POST /integrations/authentication
  - POST /customers/export
---

# Ingest users and events into MoEngage

Use this skill to push customer data into a MoEngage workspace. Every request goes to the host for the
workspace's data center — the workspace cannot be reached from another DC's host.

## 1. Resolve the host and credentials

- Base URL is `https://api-{dc}.moengage.com/v1`, where `{dc}` is `01`–`06` or `101`. Read it from the
  customer's dashboard URL (`https://dashboard-03.moengage.com` → `api-03`).
- Auth is HTTP Basic: `Authorization: Basic base64(<workspace_id>:<data_api_key>)`.
  The **Data** API key is a different key from the Push, Inform, Campaigns and Personalize keys.
- `Content-Type: application/json` on every request.
- `MOE-APPKEY: <workspace_id>` is required only for File Import APIs and the Test Connection API. Do not
  send it on Track User, Track Event, Merge, Delete, or Track Device.

Verify credentials before doing anything else with `POST /integrations/authentication`. A `401` means the
key is wrong or belongs to a different feature tile; a `403` means the feature is not enabled.

## 2. Create or update a user

`POST /customer/{app_id}` upserts a profile. Identify the user by the workspace's unique identifier
(`unique_id`/`uid`). Sending the same identifier again updates rather than duplicates.

Read a profile back with `POST /customers/export`.

## 3. Track events

`POST /event/{Workspace_ID}` records a behavioural event against a user. Events drive segmentation,
business events, flows, and campaign triggers — send them with the attributes you will later want to
filter on, because attributes cannot be back-filled onto historical events.

## 4. Register devices

`POST /device/{app_id}` creates or updates a device so push can be delivered. `POST /devices/manage`
handles opt-out and device state. Both are required for push reachability to be accurate; a segment count
returned by the platform reflects device reachability, not just profile count.

## 5. Bulk import

`POST /transition/{Workspace_ID}` is the bulk import path. Keep each payload under **100 KB** — the Bulk
Import API returns `413 Payload Too Large` above that. Batch rather than retrying a too-large body.

## Conventions and failure handling

- **Errors** use the standard envelope: `{"status":"fail","error":{"message","type","request_id"}}`.
  Log `request_id` — MoEngage support keys off it.
- **Rate limits** are per API family and surfaced as `x-ratelimit-limit`, `x-ratelimit-remaining`,
  `x-ratelimit-reset`. On `429`, back off and honour `Retry-After` where present.
- **Idempotency**: the Data APIs are keyed by the user identifier and are naturally upserting; there is
  no `Idempotency-Key` header on this family (unlike Campaigns V5 and Offerings).
- `415` means the `Content-Type` header was missing or unsupported. `405` means the wrong HTTP method.
- See `conventions/moengage-conventions.yml` and `errors/moengage-problem-types.yml`.

## Privacy

Data subject requests go through `submitGdprRequest` in `openapi/moengage-gdpr-ccpa-openapi.yml`, not
through user deletion. Use it for GDPR/CCPA erasure and access requests so the request is auditable.
