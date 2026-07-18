# ARCHITECTURE — v5 (Resilience, Authorization & Polish)

> Carried over from v4: all 8 services exist — eureka-server, api-gateway, product-service, user-service, order-service, recommendation-service, payment-service, notification-service, frontend-service — see `services-index.md` for the summary. No new services in v5, no new entities. This version hardens what already exists and closes two authorization gaps stated as known limitations since earlier versions.
> If credits are tight, this is the version it's most acceptable to do partially — sections are independent of each other, do as many as you can afford.

---

## 1. What's added (by service)

| Service | Addition |
|---|---|
| order-service | Resilience4j circuit breaker on the call to payment-service (Section 2.1) |
| recommendation-service | Resilience4j circuit breaker on calls to order-service/product-service (Section 2.2) |
| api-gateway | generic fallback route for any downstream failure (Section 2.3) |
| order-service, recommendation-service | Ownership enforcement: path `{userId}` cross-checked against JWT-derived `X-User-Id` header (Section 2.4) |
| product-service | Role-based authorization: write endpoints restricted to ADMIN role (Section 2.5) |
| all backend services | standardized error response shape + `GlobalExceptionHandler` (Section 3) |
| all backend services | `springdoc-openapi` / Swagger UI (Section 4) |
| order-service, recommendation-service, product-service | targeted tests (Section 5) |
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
| Fallback | return an empty recommendation list — this formalizes the "fail open, fail empty" behavior recommendation-service already had informally since v3; v5 just wraps it in an actual circuit breaker instead of a plain try/catch |
| Retry | one retry with a short backoff is acceptable here — these are read-only lookups, safe to retry, unlike the payment call above |

### 2.3 api-gateway fallback route

A generic fallback endpoint (e.g. `/fallback`) that any route can be configured to hit when its target circuit is open. Returns a clean JSON body: `{"status": 503, "message": "Service temporarily unavailable"}` instead of letting a raw connection-refused exception leak to the frontend as an ugly stack trace.

### 2.4 Ownership enforcement on path-scoped endpoints

v2 introduced the JWT and the gateway's `X-User-Id` header, but never cross-checked it against the `{userId}` path variable used throughout order-service and recommendation-service — a gap named explicitly in v2's Known Limitations. v5 closes it.

**The problem, precisely:** a legitimately authenticated user (a valid, correctly-issued token, no forgery or privilege escalation involved) can act on another user's resources simply by changing the `{userId}` value in the URL, since nothing checks that the path id matches the token's own identity.

**Affected endpoints:**
- order-service: `GET /api/orders/cart/{userId}`, `POST /api/orders/cart/{userId}/items`, `DELETE /api/orders/cart/{userId}/items/{itemId}`, `POST /api/orders/checkout/{userId}`, `GET /api/orders/history/{userId}`
- recommendation-service: `GET /api/recommendations/user/{userId}`

**Mechanism:** one `HandlerInterceptor` per service (not per-endpoint logic) — `preHandle` extracts `{userId}` from the request path, compares it to the `X-User-Id` header forwarded by the gateway, and throws a new `ForbiddenException` on mismatch. Register it via a `WebMvcConfigurer`, applied only to the path patterns above — same registration pattern frontend-service's `LoginRequiredInterceptor` already uses (v1 Part B), just enforcing ownership instead of presence.

| Aspect | Behavior |
|---|---|
| Trigger | `X-User-Id` header value does not match the `{userId}` path variable |
| Response | 403 Forbidden, `{"message": "You are not authorized to access this resource"}` — a new `ForbiddenException` wired into each service's existing `GlobalExceptionHandler`, same pattern as every other custom exception in the project |
| Not in scope | Role-based authorization (Section 2.5) is a separate concern — this section only fixes *ownership* (is this your own resource?), not *permission tiers* (what does your role allow you to do generally?) |

Does not touch product-service, user-service, payment-service, notification-service, api-gateway, or frontend-service — the gateway already forwards `X-User-Id` correctly (v2), and frontend-service already only ever sends the logged-in user's own id (the exploit path is hand-crafted requests via Postman/curl, not normal UI use).

### 2.5 Role-based authorization — admin-only product writes

Since v1, product-service's write endpoints (`POST /api/categories`, `POST /api/products`, `PUT /api/products/{id}`, `DELETE /api/products/{id}`, `PATCH /api/products/{id}/stock`) have been callable by any authenticated user — the `role` claim has existed on the JWT since v2 but nothing has ever read it. v5 closes this.

**The rule:** `role` must equal `"ADMIN"` for the five write endpoints listed above. `GET /api/products`, `GET /api/products/{id}`, and `GET /api/categories` remain open to any authenticated OR anonymous caller, unchanged from every prior version — this section restricts writes only, not reads.

**Mechanism — two pieces, one in the gateway, one in product-service:**

1. **api-gateway** (`JwtValidationFilter`, extended): when forwarding a request, in addition to `X-User-Id` and `X-User-Email`, also forward the JWT's `role` claim as a new header, `X-User-Role`. This is additive to the existing header-forwarding logic from v2 — no change to whitelist behavior, no change to which requests require a token at all.
2. **product-service** (new `HandlerInterceptor`, `AdminOnlyInterceptor`): registered via `WebMvcConfigurer`, applied only to the five write-endpoint path/method combinations listed above (not the GET endpoints). `preHandle` reads `X-User-Role`; if it is missing or not exactly `"ADMIN"`, throw a new `ForbiddenException` → 403, `{"message": "This action requires ADMIN privileges"}`. Same `GlobalExceptionHandler` wiring pattern as everywhere else in the project. This is a new, separate `ForbiddenException` class local to product-service — Section 2.4's `ForbiddenException` lives in order-service and recommendation-service only, not shared code.

**How a user actually becomes ADMIN in this project:** there is still no endpoint anywhere to change a user's role after registration (`role` defaults to `"CUSTOMER"` in user-service's `@PrePersist`, per v1, and nothing in v2–v4 ever offered a way to change it). For v5's purposes, promoting a test user to ADMIN is a **manual database update** — directly `UPDATE users SET role = 'ADMIN' WHERE id = ...` in Postgres, done by hand for testing. Building a proper admin-promotion flow (another admin promoting a user, or a seed/bootstrap mechanism) is explicitly out of scope for this version — name it as a known limitation, not an oversight.

**What this does not cover:** no role check is added to user-service, order-service, recommendation-service, payment-service, or notification-service — none of their endpoints are role-differentiated in this project's design (e.g. any authenticated CUSTOMER can check out their own cart, view their own history; there's no "admin views all orders" endpoint anywhere to protect). Role enforcement is scoped to product-service's writes only, because that's the only place in the whole system where an admin/customer distinction was ever meaningful per the original design.

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

Same `@RestControllerAdvice` pattern each service already has individually since v1 — v5 doesn't add new exception *types* beyond what Sections 2.4 and 2.5 introduce, it makes all six backend services agree on the same response *shape* so a client (or you, in a viva) can rely on error responses looking identical regardless of which service produced them.

---

## 4. Swagger / OpenAPI

Add `springdoc-openapi-starter-webmvc-ui` to product-service, user-service, order-service, recommendation-service, payment-service, notification-service. Each gets its own `/swagger-ui/index.html`. No custom configuration needed beyond the dependency itself and, optionally, a one-line `@OpenAPIDefinition` title/description per service.

---

## 5. Targeted tests (not full coverage)

Pick tests that prove something specific, not tests for coverage percentage's sake.

| Service | Test | What it proves |
|---|---|---|
| order-service | Checkout with insufficient stock on item 2 of a 3-item cart | v2's "validate all items before decrementing any" logic actually holds |
| order-service | Checkout where payment-service returns FAILED | v4's restock-on-failure logic correctly reverses the earlier decrement |
| order-service | Request to `/api/orders/checkout/{userId}` where `{userId}` doesn't match `X-User-Id` | Section 2.4's ownership interceptor returns 403 and never reaches the checkout logic |
| recommendation-service | A known small dataset run through the v4 scoring formula | the score isn't hand-waved — assert the exact expected ranking against a formula you can also compute by hand |
| product-service | `POST /api/products` with `X-User-Role: CUSTOMER` | Section 2.5's admin-only interceptor returns 403 and never reaches the controller |
| product-service | `POST /api/products` with `X-User-Role: ADMIN` | the same interceptor correctly allows the request through when the role is right |
| api-gateway | An authenticated-route request with no `Authorization` header | returns 401 and never reaches the downstream service |

Tests like these, each provably tied to a specific design decision from an earlier version file (or this one), are worth more in a viva than broad but shallow coverage across everything.

---

## 6. Known-limitations writeup — out of scope for this file

The consolidated known-limitations writeup (pulling together every "Known limitations" section from `architecture-v1.md` through this file) is tracked in `architecture-master-methodology.md`, Section 11 — a human-only file, never pasted into Copilot/Amazon Q. This section is intentionally left as a pointer only, so nothing in this AI-facing file is addressed to a human reader rather than the model being prompted with it.

Two items worth flagging for that writeup specifically, since this version resolves one and partially resolves another:
- The `userId`-vs-JWT ownership gap named in `architecture-v2.md` is closed as of Section 2.4 above — do not carry it forward as unresolved.
- Role-based authorization is now enforced for product-service writes (Section 2.5), but the *mechanism to grant the ADMIN role itself* remains manual (direct database edit, no promotion endpoint) — this residual gap should still be named.

---

## 7. Testing checklist before calling v5 — and the whole project — "done"

- [ ] Manually stop payment-service, attempt a checkout, confirm order lands in `PAYMENT_PENDING_RETRY` (not `PAYMENT_FAILED`, not a raw 500)
- [ ] Manually stop order-service or product-service, hit a recommendation endpoint, confirm it returns an empty list, not an error
- [ ] Every backend service's error responses match the shape in Section 3 exactly, for at least one triggered error per service
- [ ] All six backend services' Swagger UIs load and show accurate endpoint lists
- [ ] Log in as user A, obtain their JWT, attempt `GET /api/orders/cart/{userB's id}` using user A's token — confirm 403, not 200
- [ ] Same check repeated for checkout, history, and recommendation-service's user endpoint — confirm all five affected endpoints reject a mismatched userId with 403
- [ ] Confirm a matching userId (own resource) still returns 200 as before — the ownership interceptor must not break the normal, correct-owner case
- [ ] Manually promote a test user to ADMIN via direct SQL update; confirm that user CAN create/update/delete products and adjust stock
- [ ] Confirm a CUSTOMER-role user gets 403 attempting the same five write endpoints
- [ ] Confirm GET endpoints on product-service still work for both roles, and for anonymous/no-token requests, unaffected by Section 2.5
- [ ] All tests in Section 5 pass
- [ ] The known-limitations writeup exists in `architecture-master-methodology.md` Section 11 and covers every item listed there, including the two new v5-specific notes in Section 6 above
- [ ] Full end-to-end walkthrough (register → browse → view products → cart → checkout with a real payment success → checkout with a forced payment failure → recommendations reflect purchase/view history → admin-only product write correctly blocked for a customer, correctly allowed for an admin) works with zero unhandled errors anywhere in the chain