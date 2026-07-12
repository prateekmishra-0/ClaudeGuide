# ARCHITECTURE — v5 (Resilience & Polish)

> Carried over from v4: all 8 services exist — eureka-server, api-gateway, product-service, user-service, order-service, recommendation-service, payment-service, notification-service, frontend-service — see `services-index.md` for the summary. No new services in v5, no new entities, no new business features. This version hardens what already exists.
> If credits are tight, this is the version it's most acceptable to do partially — sections are independent of each other, do as many as you can afford.

---

## 1. What's added (by service)

| Service | Addition |
|---|---|
| order-service | Resilience4j circuit breaker on the call to payment-service (Section 2) |
| recommendation-service | Resilience4j circuit breaker on calls to order-service/product-service (Section 2) |
| api-gateway | generic fallback route for any downstream failure (Section 2) |
| all backend services | standardized error response shape + `GlobalExceptionHandler` (Section 3) |
| all backend services | `springdoc-openapi` / Swagger UI (Section 4) |
| order-service, recommendation-service | targeted tests (Section 5) |
| api-gateway | one security test (Section 5) |

---

## 2. Resilience4j — circuit breakers

Add `spring-boot-starter-actuator` + `resilience4j-spring-boot3` to the services below. Not every inter-service call gets one — only the two identified as actually risky.

### 2.1 order-service → payment-service

| Aspect | Behavior |
|---|---|
| Trigger | payment-service times out or is unreachable (not: payment-service responds with `FAILED` — that's a normal business outcome from v4, already handled, not a circuit-breaker case) |
| Fallback | order status set to `PAYMENT_PENDING_RETRY` (new status value, distinct from v4's `PAYMENT_FAILED`) — a timeout is not the same as a rejection, and shouldn't trigger the v4 restock logic, since the payment may have actually succeeded on payment-service's side despite the timeout |
| Retry | none on this call — retrying a payment blindly risks double-processing; since v4's mock payment logic doesn't move real money this is a low-stakes gap, but state it explicitly as a known limitation rather than silently retrying |

### 2.2 recommendation-service → order-service / product-service

| Aspect | Behavior |
|---|---|
| Trigger | either upstream service times out or is unreachable |
| Fallback | return an empty recommendation list — this formalizes the "fail open, fail empty" behavior recommendation-service already had informally since v3 (Section 2.5 of `architecture-v3.md`); v5 just wraps it in an actual circuit breaker instead of a plain try/catch |
| Retry | one retry with a short backoff is acceptable here — these are read-only lookups, safe to retry, unlike the payment call above |

### 2.3 api-gateway fallback route

A generic fallback endpoint (e.g. `/fallback`) that any route can be configured to hit when its target circuit is open. Returns a clean JSON body: `{"status": 503, "message": "Service temporarily unavailable"}` instead of letting a raw connection-refused exception leak to the frontend as an ugly stack trace.

---

## 3. Standardized error response shape

Every backend service (product, user, order, recommendation, payment, notification) gets a `GlobalExceptionHandler` returning this exact shape for every error:

```json
{
  "timestamp": "2026-07-15T10:23:00",
  "status": 404,
  "error": "Not Found",
  "message": "Product with id 42 not found",
  "path": "/api/products/42"
}
```

| Field | Source |
|---|---|
| timestamp | `LocalDateTime.now()` at the moment the handler runs |
| status | the HTTP status code being returned |
| error | the standard HTTP reason phrase for that status |
| message | exception-specific, human-readable |
| path | `HttpServletRequest.getRequestURI()` |

Same `@RestControllerAdvice` pattern each service already has individually since v1 (each service's own `GlobalExceptionHandler` for its own custom exceptions) — v5 doesn't add new exception types, it makes all six services agree on the same response *shape* so a client (or you, in a viva) can rely on error responses looking identical regardless of which service produced them.

---

## 4. Swagger / OpenAPI

Add `springdoc-openapi-starter-webmvc-ui` to product-service, user-service, order-service, recommendation-service, payment-service, notification-service — same dependency your reference guide already uses for the book-partner-portal. Each gets its own `/swagger-ui/index.html`. No custom configuration needed beyond the dependency itself and, optionally, a one-line `@OpenAPIDefinition` title/description per service so they're distinguishable when you have six tabs open.

---

## 5. Targeted tests (not full coverage)

Pick tests that prove something specific, not tests for coverage percentage's sake.

| Service | Test | What it proves |
|---|---|---|
| order-service | Checkout with insufficient stock on item 2 of a 3-item cart | v2's "validate all items before decrementing any" logic actually holds — nothing gets decremented if any item fails |
| order-service | Checkout where payment-service returns FAILED | v4's restock-on-failure logic correctly reverses the earlier decrement |
| recommendation-service | A known small dataset (e.g. 3 products, fixed purchase/view counts) run through the v4 scoring formula | the score isn't hand-waved — assert the exact expected ranking against a formula you can also compute by hand |
| api-gateway | An authenticated-route request with no `Authorization` header | returns 401 and never reaches the downstream service (verify via a mock/stub or by checking the downstream service's logs show no incoming request) |

Three or four tests like these, each provably tied to a specific design decision from an earlier version file, are worth more in a viva than broad but shallow coverage across everything.

---

## 6. Not code: the "known limitations" writeup

Write this yourself, not via Copilot/Amazon Q — it belongs in your top-level `architecture.md` (the human-only master file), not in this AI-facing file. Pull together every "Known limitations" section from `architecture-v1.md` through this file into one honest paragraph or table covering: no message broker (direct REST instead of events), no saga/distributed-transaction framework (manual compensation in order-service instead), no real payment gateway, single Postgres instance with per-service schemas instead of fully separate database servers, no refresh tokens, no role enforcement despite the JWT carrying a role claim. Frame each as a stated trade-off with a reason, not an accident — that's the difference this whole file structure has been building toward.

---

## 7. Testing checklist before calling v5 — and the whole project — "done"

- [ ] Manually stop payment-service, attempt a checkout, confirm order lands in `PAYMENT_PENDING_RETRY` (not `PAYMENT_FAILED`, not a raw 500)
- [ ] Manually stop order-service or product-service, hit a recommendation endpoint, confirm it returns an empty list, not an error
- [ ] Every backend service's error responses match the shape in Section 3 exactly, for at least one triggered error per service
- [ ] All six backend services' Swagger UIs load and show accurate endpoint lists
- [ ] The four tests in Section 5 pass
- [ ] The known-limitations writeup exists in `architecture.md` and covers every item listed in Section 6
- [ ] Full end-to-end walkthrough (register → browse → view products → cart → checkout with a real payment success → checkout with a forced payment failure → recommendations reflect purchase/view history) works with zero unhandled errors anywhere in the chain
