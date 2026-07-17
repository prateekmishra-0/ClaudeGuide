# ARCHITECTURE — v3 (Recommendation Engine v1)

> Carried over from v2: eureka-server, api-gateway, product-service, user-service, order-service, frontend-service all unchanged from v2 — see `services-index.md` for the summary. This file covers only the new recommendation-service and the small frontend/gateway additions needed to surface it.
> New in v3: recommendation-service (new service, no database). Rule-based only — co-occurrence counting and category fallback. No ML, no weighting, no view tracking (that's v4).

---

## 1. Services in this version

| Service | Port | Change from v2 |
|---|---|---|
| eureka-server | 8761 | unchanged |
| api-gateway | 8000 | one new route added (Section 3) |
| product-service | 8081 | unchanged |
| user-service | 8082 | unchanged |
| order-service | 8083 | unchanged |
| recommendation-service | 8084 | **new** |
| frontend-service | 8080 | two new sections added to two templates (Section 5) |

No new entities anywhere. recommendation-service is deliberately stateless — it owns no Postgres schema, no tables. Every call recomputes from live data fetched from order-service and product-service. This is a stated v3 simplification (see Known Limitations), not an oversight.

---

## 2. recommendation-service

### 2.1 What it calls (inter-service dependencies)

| Calls | Endpoint | Why |
|---|---|---|
| order-service | `GET /api/orders/history/{userId}` (existing v1 endpoint) | to get a user's past orders for the user-level recommendation |
| order-service | **new**: `GET /api/orders/containing-product/{productId}` | returns all PLACED orders that include this product — needed for co-occurrence counting; order-service needs this new endpoint added (Section 2.4) |
| product-service | `GET /api/products?categoryId={id}` (existing v1 endpoint) | for the category-fallback rule |
| product-service | `GET /api/products/{id}` (existing v1 endpoint) | to resolve a product's category before running the fallback rule |

### 2.2 Algorithm — Rule 1: frequently bought together

For a given `productId`:
1. Call order-service's new `containing-product` endpoint → get all PLACED orders that include this product
2. From those orders' item lists, tally every *other* `productId` that appears across them — a simple frequency count, e.g. `Map<Long productId, Integer count>`
3. Sort descending by count, exclude the original `productId` itself, take the top N (N = 4, hardcode this — no config needed for v3)

This requires zero math beyond counting occurrences in a map. If two products have the same count, order is whatever the map iteration gives you — no tie-breaking logic needed in v3.

### 2.3 Algorithm — Rule 2: category fallback

Triggered only when Rule 1 returns fewer than N results (new product with no order history yet, or early-stage low order volume):
1. Resolve the original product's `categoryId` via product-service
2. Fetch other products in that category (`GET /api/products?categoryId={id}`), excluding the original product itself
3. Fill in the remaining slots (up to N total, combining with whatever Rule 1 already found) — no ranking signal beyond "same category," insertion/return order from product-service is fine as-is

### 2.4 New endpoint required on order-service

| Method | Path | Purpose | Notes |
|---|---|---|---|
| GET | `/api/orders/containing-product/{productId}` | list all PLACED orders whose items include this productId | new for v3 — a query across all orders' items, filtered by productId and status=PLACED |

This is the one schema-adjacent change in v3, and it lives in order-service, not recommendation-service — worth calling out explicitly in your order-service prompt for this version so it isn't missed.

### 2.5 recommendation-service's own endpoints

| Method | Path | Purpose | Notes |
|---|---|---|---|
| GET | `/api/recommendations/product/{productId}` | Rule 1 + Rule 2 fallback for a single product | powers the product detail page's "you might also like" |
| GET | `/api/recommendations/user/{userId}` | recommendations for a logged-in user | pulls the user's most recent PLACED order, runs Rule 1+2 per product in it, merges/dedupes results, excludes products the user already owns |

Both endpoints return a flat list of product IDs (or full product summaries — resolve name/price/image via product-service before returning, so the frontend doesn't need a second round-trip). If either upstream call (order-service or product-service) fails or returns nothing usable, return an empty list — never a 500. A broken recommendation is a missing widget, not a broken page (this principle carries forward into v5's circuit-breaker fallback, which formalizes this same "fail open, fail empty" behavior).

---

## 3. api-gateway change

| Route ID | Predicate | Destination |
|---|---|---|
| recommendation-route | `Path=/api/recommendations/**` | `lb://recommendation-service` |

Not uniformly authenticated — split by path and method, same pattern v2 established for product/category GETs:
- `GET /api/recommendations/product/{productId}` — **public, add to the gateway's whitelist**, not the authenticated set. This powers the "you might also like" section on product-detail.html, which is a non-login-gated page per v1 — gating this behind a token would silently break recommendations for every anonymous visitor, the same contradiction v2's original (uncorrected) filter spec had with catalog browsing.
- `GET /api/recommendations/user/{userId}` — **authenticated, stays in the JWT-required set**. This is already only ever called from a login-gated context (home.html's "Recommended for you" section, rendered only for logged-in users per Section 5), so no whitelist entry is needed or correct here.

---

## 4. product-service / order-service changes

- product-service: no changes.
- order-service: one new endpoint only (Section 2.4). No entity changes — this is a new query method on the existing `OrderItem`/`Order` repositories, not new data.

---

## 5. frontend-service changes

| Page | Addition | Calls |
|---|---|---|
| `product-detail.html` | "You might also like" section | `GET /api/recommendations/product/{id}` through the gateway |
| `home.html` | "Recommended for you" section — **only rendered if the user is logged in** | `GET /api/recommendations/user/{userId}` through the gateway |

Logged-out users see the plain v1 product grid on the homepage with no recommendation call made at all — there's no order history to work from, and calling the endpoint anyway would just return an empty list for no benefit.

---

## 6. Known limitations (intentional, fixed in later versions)

- No weighting — a co-occurrence count of 2 ranks above a count of 1 with no other signal considered (v4 adds view-based weighting)
- No view tracking exists yet — recommendations are purchase-history-only (v4)
- recommendation-service recomputes from scratch on every call, no caching — acceptable at demo data volumes, would need revisiting at real scale
- Tie-breaking in Rule 1 is undefined (whatever order the map gives) — not worth solving for a project of this size

---

## 7. Testing checklist before calling v3 "done"

- [ ] order-service's new `containing-product` endpoint returns correct orders when tested directly (Postman)
- [ ] A product with real order history (place a few test orders containing it first) returns sensible Rule 1 results via `/api/recommendations/product/{id}`
- [ ] A brand-new product with zero order history falls back correctly to Rule 2 (same-category results)
- [ ] `/api/recommendations/user/{userId}` returns results for a user with at least one PLACED order, and doesn't recommend a product the user already bought
- [ ] Manually break product-service or order-service (stop it) and confirm recommendation-service's endpoints return an empty list, not a 500
- [ ] `GET /api/recommendations/product/{id}` works through the gateway with NO token present — confirm the "you might also like" section renders for a logged-out visitor on product-detail.html
- [ ] `GET /api/recommendations/user/{userId}` without a token returns 401 from the gateway
- [ ] Frontend: product detail page shows the "you might also like" section (logged in AND logged out); homepage shows "recommended for you" only when logged in
