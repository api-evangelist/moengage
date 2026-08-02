---
name: Manage MoEngage catalogs, recommendations, coupons and offers
description: Load a product catalog, drive recommendations from it, administer coupon lists and files, and create offerings for Offer Decisioning under the Idempotency-Key contract.
api: openapi/moengage-catalog-openapi.yml
operations:
  - createCatalog
  - addCatalogAttributes
  - ingestCatalogItems
  - updateCatalogItems
  - deleteCatalogItems
  - getItemDetails
  - fetchRecommendationMetadata
  - fetchRecommendationResults
  - createCouponList
  - uploadCouponFile
  - activateCouponList
  - generateUsageReport
  - createPublicOffering
  - updatePublicOffering
  - listPublicOfferings
---

# Manage MoEngage catalogs, recommendations, coupons and offers

## 1. Catalog

`createCatalog` defines the catalog, `addCatalogAttributes` extends its schema, then `ingestCatalogItems`
loads rows. `updateCatalogItems` and `deleteCatalogItems` maintain it; `getItemDetails` reads back.

Limits that bite:
- **5 MB max payload.** `413` from `createCatalog` also signals the attribute limit was exceeded.
- **100 requests/min or 1,000/hour.** Chunk large loads and pace them.

Add every attribute you will later want to filter or template on — recommendations and campaign content
can only reference attributes the catalog actually carries.

## 2. Recommendations

`fetchRecommendationMetadata` returns the configuration for a recommendation ID;
`fetchRecommendationResults` returns the items. Payloads cap at **1 MB**. Recommendations resolve against
catalog items, so a stale catalog produces stale recommendations.

## 3. Coupons

A coupon list is a container; coupon files are the code batches inside it.

1. `createCouponList`
2. `uploadCouponFile` — one or more code files
3. `activateCouponList` — the list is not usable in campaigns until activated
4. `generateUsageReport` — redemption reporting
5. `archiveCouponList` when retired

`fetchAllCouponLists`, `fetchCouponList`, `fetchAllCouponFiles`, `fetchCouponFile` and `deleteCouponFile`
round out the surface.

## 4. Offer Decisioning

`createPublicOffering` / `updatePublicOffering` require an `Idempotency-Key` header (**UUID v4**) with a
**24-hour** replay window: the same key inside 24 hours returns the original response without re-running.

Error codes specific to this API:

| Code | HTTP | Meaning |
| --- | --- | --- |
| `VALIDATION_FAILED` | 400 | Check `error.details[]` — all violations are returned at once |
| `INVALID_OFFER_NAME` | 400 | `name` must be letters, numbers and underscores only |
| `DUPLICATE_OFFER_NAME` | 400 | A non-archived offering already uses that name |
| `INVALID_SCHEDULING` | 400 | `expiry_datetime` must be after `start_datetime` and not in the past |
| `IMMUTABLE_FIELD` | 400 | The field is status-locked |
| `IDEMPOTENCY_CONFLICT` | 409 | Same key, different body — use a new UUID v4 |
| `EXPIRED_OFFERING` / `ARCHIVED_OFFERING` | 403 | Not modifiable |
| `OFFER_NOT_FOUND` | 404 | No such `offer_id` in this workspace |

`429` carries a `Retry-After` in seconds. `503` is transient — retry with exponential backoff.

`listPublicOfferTemplates` returns the templates an offering can be built from.

## Conventions

Offerings uses the v5 envelope (`response_id` + `error.code`/`target`/`details`/`doc_url`); catalog,
recommendations and coupons use the standard envelope. See `conventions/moengage-conventions.yml`.
