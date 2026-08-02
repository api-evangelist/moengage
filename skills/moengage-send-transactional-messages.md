---
name: Send MoEngage transactional push and alerts
description: Deliver transactional push notifications and multi-channel Inform alerts, including correct use of the Inform sandbox host and its separate Test Alert ID.
api: openapi/moengage-push-v2-1-openapi.yml
operations:
  - send_push_v2_1
  - POST /transaction/sendpush
  - POST /alerts/send
---

# Send MoEngage transactional push and alerts

Two distinct surfaces, on two distinct hosts, with two distinct API keys.

## Transactional push

- Host: `https://pushapi-{dc}.moengage.com` (**not** `api-{dc}`).
- Spec: `openapi/moengage-push-v2-1-openapi.yml` (`send_push_v2_1`) and the older
  `openapi/moengage-push-openapi.yml` (`POST /transaction/sendpush`).
- Auth: Basic with the **Push** API key from Settings → Account → APIs.
- Prefer `send_push_v2_1`; the v2 path remains for existing integrations.

The recipient must have a registered device — see the ingest skill and `POST /device/{app_id}`. A send to
an unreachable user is not an error at the API layer; check delivery stats, not the HTTP status.

## Inform — unified transactional alerts

`POST /alerts/send` (`openapi/moengage-inform-openapi.yml`) sends one alert across SMS, Email and Push
from a single call. Auth uses the **Inform** API key.

**Inform is the only MoEngage surface with a real sandbox:**

| | Live | Test |
| --- | --- | --- |
| Host | `https://api-0{dc}.moengage.com` | `https://sandbox-api-0{dc}.moengage.com` |
| Identifier | Alert ID | **Test Alert ID** |
| Logs | Alert Info section | Step 3 of Alert Creation |

Every alert has **two** IDs. Sending the live Alert ID to the sandbox host returns
`Live Alert Request received on Sandbox API endpoint`. Pick up the Test Alert ID by editing the alert and
going to page 3. Alert logs are retained for 30 days.

## Conventions

- Errors use the standard envelope `{"status":"fail","error":{"message","type","request_id"}}`.
- These endpoints do **not** take an `Idempotency-Key` header — that contract exists on Campaigns V5 and
  Offerings only. Build your own dedupe key upstream if you retry sends.
- Honour `x-ratelimit-*` headers and back off on `429`.
- See `conventions/moengage-conventions.yml` and `sandbox/moengage-sandbox.yml`.
