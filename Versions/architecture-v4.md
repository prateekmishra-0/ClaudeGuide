# ARCHITECTURE — v4 (Recommendation v2 + Payment + Notifications)

> Carried over from v3: eureka-server, api-gateway, product-service, user-service, order-service, recommendation-service, frontend-service all exist — see `services-index.md` for the summary. This file covers what's new or changed in v4: view tracking + weighted scoring in recommendation-service, a new payment-service, a new notification-service, and the resulting checkout flow change in order-service.
> This is the busiest version — expect it to take the largest share of your remaining credits.

---

## 1. Services in this version

| Service | Port | Change from v3 |
|---|---|---|
| eureka-server | 8761 | unchanged |
| api-gateway | 8000 | two new routes (Section 5) |
| product-service | 8081 | **new entity**: ProductView (Section 2) |
| user-service | 8082 | unchanged |
| order-service | 8083 | checkout flow expanded to 3 states, calls payment-service (Section 4) |
| recommendation-service | 8084 | scoring logic upgraded, calls the new view-tracking data (Section 3) |
| payment-service | 8085 | **new** |
| notification-service | 8086 | **new** |
| frontend-service | 8080 | view-tracking call added, checkout page reflects payment result |

---

## 2. product-service changes

### 2.1 New entity: ProductView

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-generated (IDENTITY) |
| userId | Long | `@NotNull` — reference only, no cross-schema FK |
| productId | Long | `@ManyToOne` to Product, `@JoinColumn`, not nullable |
| viewedAt | LocalDateTime | `@PrePersist` |

3 fields plus the relationship. No uniqueness constraint — the same user viewing the same product twice creates two rows; this is intentional, view *frequency* is part of what recommendation-service will count in Section 3.

### 2.2 New endpoint

| Method | Path | Purpose | Notes |
|---|---|---|---|
| POST | `/api/products/{id}/view` | record a view | body: `{"userId": ...}`. Fire-and-forget from the frontend — return 201 with no meaningful body. No auth requirement beyond the standard gateway JWT check; if you want to allow anonymous views to no-op silently instead of 401ing, that's a valid choice too — pick one and be consistent, it's a minor call either way |
| GET | `/api/products/{id}/views/count-by-viewer` | returns `Map<userId, viewCount>` for this product | used by recommendation-service's scoring in Section 3 — a plain `GROUP BY` query, not complex |

---

## 3. recommendation-service — v2 scoring

v3's Rule 1 (pure co-occurrence count from purchases) stays exactly as it was — it's not replaced, it's one input to a combined score now.

### 3.1 New signal: co-view count

For two products X and Y, `co_view_count` = number of distinct users who have viewed both X and Y (via `ProductView` rows, queried through product-service's new endpoints). Computed the same shape as v3's co-purchase tally: fetch viewers of X, fetch viewers of Y (or fetch all viewers of X's category cohort — simplest correct v4 approach is: for each candidate product Y already surfaced by v3's co-occurrence/category rules, separately check how many users viewed both X and Y), intersect, count.

### 3.2 Combined scoring formula

```
score(Y | X) = (co_purchase_count(X, Y) × 3) + (co_view_count(X, Y) × 1)
```

Purchases weighted 3x over views — a purchase is a stronger intent signal than a click. This ratio is a stated design choice, not tuned against real data (there isn't enough of it at your project's scale to tune meaningfully) — say exactly that if asked to justify the "3."

`GET /api/recommendations/product/{productId}` now ranks by this combined score instead of raw co-purchase count. Category fallback (v3's Rule 2) is unchanged, still used to fill remaining slots when the combined-score list has fewer than N results.

### 3.3 Category-affinity boost for user-level recommendations

`GET /api/recommendations/user/{userId}`:
1. From the user's order history (existing v1 endpoint), tally which categories they've purchased from most — top 2 categories by purchase count
2. For each candidate product produced by the existing per-product logic (Section 3.2), add a flat `+2` to its score if that product's category is in the user's top 2

```
score(Y | userU) = score(Y | most_relevant_purchased_product) + (2 if Y.category ∈ topCategories(userU) else 0)
```

Same principle as 3.2 — simple, explainable, no ML.

---

## 4. payment-service (new)

### 4.1 Entity

**Payment**

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-generated (IDENTITY) |
| orderId | Long | `@NotNull` — reference only |
| amount | BigDecimal | `@NotNull`, `@DecimalMin("0.01")` |
| status | String | `SUCCESS` or `FAILED` |
| createdAt | LocalDateTime | `@PrePersist` |

4 fields.

### 4.2 Mock processing logic

`POST /api/payments` — body: `{"orderId": ..., "amount": ...}`. Deterministic mock, no real gateway:
- Reject (`FAILED`) if `amount <= 0` — a real validation, not a fake one
- Reject (`FAILED`) if the amount's cents value ends in `.13` (arbitrary but deterministic and testable — avoids `Math.random()` so tests are reproducible; document this rule plainly in code comments so it doesn't look like a mystery bug)
- Otherwise `SUCCESS`
- Persist the attempt either way, return `{"status": "...", "paymentId": ...}`

### 4.3 Endpoint

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/payments` | process a mock payment for an order |
| GET | `/api/payments/order/{orderId}` | look up a payment's result by order — used by order-service if it needs to re-check status |

---

## 5. api-gateway changes

| Route ID | Predicate | Destination |
|---|---|---|
| payment-route | `Path=/api/payments/**` | `lb://payment-service` |
| notification-route | `Path=/api/notifications/**` | `lb://notification-service` — only needed if you expose a manual trigger/lookup; if notification-service is purely called service-to-service from order-service with no external route, this entry is optional |

Both require a valid JWT, same as everything since v2.

---

## 6. order-service changes — checkout flow, now 3 states

`Order.status` gains two new valid values: `PENDING_PAYMENT`, `PAID`, `PAYMENT_FAILED` (in addition to v1's `CART`, `PLACED` — note `PLACED` is effectively superseded by `PAID`/`PAYMENT_FAILED` going forward; keep `PLACED` in the codebase for backward compatibility with any v1/v2/v3 test data, but new checkouts never land there again).

```
POST /api/orders/checkout/{userId}
  1. (unchanged from v2) validate stock for all items, decrement stock for all items
  2. Create the order with status PENDING_PAYMENT
  3. Call payment-service: POST /api/payments {orderId, amount: order total}
  4a. SUCCESS → order status → PAID → call notification-service (Section 7) with a success message
  4b. FAILED  → order status → PAYMENT_FAILED
              → for EVERY item in the order, call product-service PATCH /api/products/{id}/stock
                with a POSITIVE delta equal to the quantity (restock what step 1 decremented)
              → call notification-service with a failure message
  5. Return the order with its final status
```

The restock-on-failure step in 4b is the one piece of real compensation logic in v4 — it's a simplified, synchronous version of what a saga pattern would do properly in a real distributed system with async messaging. Worth stating exactly that in your prompt to Copilot, and worth stating exactly that in your own "known limitations" writeup too — it's good, honest engineering framing.

---

## 7. notification-service (new)

No entity, no database. A single endpoint called synchronously by order-service, not event-driven (no message broker available given the no-Docker constraint — this is a stated tradeoff, see Known Limitations).

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/notifications` | body: `{"orderId": ..., "message": "..."}` — logs the message (console/log file). No email/SMS in the baseline v4 build. |

If you have spare time near the end of the project, swapping the log line for a real `JavaMailSender` call with a throwaway SMTP account is optional polish — not required for v4 to be "done."

---

## 8. frontend-service changes

- Product detail page fires `POST /api/products/{id}/view` on page load, for logged-in users only, fire-and-forget (don't block rendering on this call's response)
- Checkout confirmation page now shows the actual payment outcome (`PAID` vs `PAYMENT_FAILED`) instead of the old generic "order placed" message from v1/v2/v3

---

## 9. Known limitations (intentional, fixed in later versions)

- notification-service is called synchronously via direct REST, not through an event bus — a real message broker (Kafka/RabbitMQ) is the textbook-correct choice, ruled out here because Docker isn't available; this is a known, deliberate trade-off, not an oversight
- Restock-on-payment-failure is the only compensation logic in the system — a crash between steps in the checkout flow (e.g. server dies after decrementing stock but before calling payment-service) leaves the order stuck in an inconsistent state with no automatic recovery; acceptable to leave unsolved and name explicitly in your final writeup
- No retry on a failed payment call itself (a payment-service timeout is treated the same as a payment-service rejection) — v5's circuit breaker section addresses this distinction
- Payment mock logic is deterministic but arbitrary (the `.13` cents rule) — fine for testing, obviously not realistic

---

## 10. Testing checklist before calling v4 "done"

- [ ] `POST /api/products/{id}/view` records rows; `GET /api/products/{id}/views/count-by-viewer` returns correct counts
- [ ] Recommendation scores visibly change when you manually generate extra views for a product (compare before/after)
- [ ] A payment amount that triggers the `.13` rule returns FAILED, and the corresponding order ends up in `PAYMENT_FAILED` with stock correctly restocked
- [ ] A normal payment amount returns SUCCESS, order ends up `PAID`, stock stays decremented (not restocked)
- [ ] notification-service log output shows a message for both the success and failure paths
- [ ] Full frontend walkthrough: view a few products, add to cart, checkout, confirm the payment outcome displays correctly
