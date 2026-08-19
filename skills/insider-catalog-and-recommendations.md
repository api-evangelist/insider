---
name: insider-catalog-and-recommendations
description: Keep the Insider One product catalog current and serve recommendations or search results from it.
api: Insider One Catalog / Recommendation / Eureka APIs
spec: openapi/insider-catalog-openapi.yml
base_url: https://catalog.api.useinsider.com
operations:
  - addNewProductsInAFlatFormat
  - addNewProductsInANestedFormat
  - updateExistingProductsInAFlatFormat
  - updateExistingProductsInANestedFormat
  - createLocaleConfigurations
  - getRecommendations
  - getSearchResults
  - collectUserEvent
generated: '2026-08-13'
method: generated
source: openapi/insider-catalog-openapi.yml + openapi/insider-recommendation-openapi.yml + openapi/insider-eureka-search-openapi.yml
---

# Catalog, recommendations and search

## Catalog writes — mind the shared 60/minute budget

`POST https://catalog.api.useinsider.com/v2/ingest` (new products) and `/v2/update` (existing), each
with a `/nested` variant for hierarchical payloads. The body is a JSON **array** of products.

**All Catalog API endpoints share one 60 requests/minute budget.** That is the tightest ceiling in
the estate relative to its job, so batch: send arrays, do not loop per product. Over-size payloads
return `400 {"success": false, "message": "Maximum allowed size is 100."}` or
`Maximum allowed record count is exceeded.`

Product fields observed in Insider One's own examples: `item_id`, `locale`, `name`, `url`,
`image_url`, `category[]`, `brand`, `color`, `groupcode`, `price` and `original_price` as
currency-keyed maps (`{"USD": 129.99}`), `in_stock`, `stock_count`.

The primary key is **`item_id` + `locale`** — the same product exists once per locale. `locale` can be
compound (`en_US:newyork`) to model store-level inventory. Register locales first with
`createLocaleConfigurations` (`POST /v2/locales/batch`, `{language_and_country_code, store_id}`).
`groupcode` groups variants of one product.

## Recommendations — one path, twenty algorithms

`GET https://recommendation.api.useinsider.com/v2/{algorithm-name}`. The algorithm is the path
segment. Published algorithms: `similar`, `complementary`, `substitute`, `visually-similar`,
`purchased-together`, `viewed-together`, `last-purchased-together`, `recently-viewed`, `top-sellers`,
`trending`, `most-popular`, `most-valuable`, `new-arrivals`, `highest-discounted`, `user-based`,
`user-engagement`, `mixed`, `manual-merchandising`, `chef`.

All endpoints share **1,000 calls/minute** across the whole Recommendation API. Cache on your side;
this is a page-render dependency on a pooled budget.

Recommendation quality depends on catalog freshness and on event streaming through
`upsertUserData` — see `skills/insider-ingest-user-data.md`.

## Search (Eureka)

- `GET https://ineureka.api.useinsider.com/api/web/search` — results plus facets.
- `GET https://ineureka.api.useinsider.com/api/web/collections` and `/collections/{type}` — category
  and brand merchandising, taking `cf` (collection filter), `p` (partner id), `l` (locale),
  `c` (currency).
- Type-ahead suggestions are served from a **per-customer host**:
  `GET https://{domain_name}/api/web/suggestions/query`. Insider One issues that hostname; do not
  guess it from your domain.

## Close the loop — `collectUserEvent`

`POST https://eurekaevent.api.useinsider.com/api/v1/events` carries the five interaction types
Eureka ranks on: search, product click, product list view, add to cart, purchase. Search relevance
degrades without them, so send them from the same page that renders the results.

## Errors

Eureka uses its own envelope:
`{"status": "Invalid|BusinessException|Error", "data": null, "error": {"code": "...", "message": "..."},
"validations": [...]}`. Catalog uses `{"success": false, "message": "..."}`. Branch per host — see
`errors/insider-problem-types.yml`.
