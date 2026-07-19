# ARCHITECTURE — v5 (Resilience, Authorization & Admin Panel)

> Carried over from v4: all 8 services exist — eureka-server, api-gateway, product-service, user-service, order-service, recommendation-service, payment-service, notification-service, frontend-service — see `services-index.md` for the summary. No new services in v5, no new entities. This version hardens what already exists, closes two authorization gaps stated as known limitations since earlier versions, and adds an admin panel on top of the role system it introduces.
> If credits are tight, Sections 2.1–2.5 (resilience, ownership, roles) are independent of Sections 3–5 (admin panel) and can be done as separate passes — but Sections 3–5 depend on Section 2.4 (patched) and Section 2.5 (role forwarding) already being built, since the admin panel has no security boundary without them. Build in section order.

---

## 1. What's added (by service)

| Service | Addition |
|---|---|
| order-service | Resilience4j circuit breaker on the call to payment-service (Section 2.1) |
| recommendation-service | Resilience4j circuit breaker on calls to order-service/product-service (Section 2.2) |
| api-gateway | generic fallback route for any downstream failure (Section 2.3) |
| order-service, recommendation-service | Ownership enforcement: path `{userId}` cross-checked against JWT-derived `X-User-Id` header, with an admin bypass on order-service (Section 2.4) |
| product-service | Role-based authorization: write endpoints restricted to ADMIN role (Section 2.5) |
| order-service | Admin-only restriction on `GET /api/orders/containing-product/{productId}` when reached via the gateway (Section 2.6) |
| user-service | Admin-only user listing, lookup, and deletion endpoints (Section 3) |
| frontend-service | Full admin panel — product/category/stock management, user list and deletion, per-product buyer view (Section 5, 5.4a) |
| frontend-service | Full admin panel — product/category/stock management, user list and deletion (Section 5) |
| all backend services | standardized error response shape + `GlobalExceptionHandler` (Section 6) |
| all backend services | `springdoc-openapi` / Swagger UI (Section 7) |
| order-service, recommendation-service, product-service, user-service | targeted tests (Section 8) |
| api-gateway | one security test (Section 8) |

---

## 2. Resilience4j — circuit breakers, ownership, and roles

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

A generic fallback endpoint (e.g. `/fallback`) that any route can be configured to hit when its target circuit is open. Returns a clean JSON body: `{"status": 503, "message": "Service temporarily unavailable"}` instead of letting a raw connection-refused exception leak to the frontend as an ugly stack trace. Whatever the v5 prompt lands on for wiring this — a `CircuitBreaker` filter added to an existing route entry, or a plain local `@RestController` endpoint the filter forwards to — it must use the webmvc-package `CircuitBreaker` filter (`org.springframework.cloud.gateway.server.mvc.filter`), not the reactive one, and any filter config still lives under `spring.cloud.gateway.server.webmvc.routes`, consistent with every other route since v2. This section is under-specified on purpose (the exact mechanism isn't chosen yet) — resolve the implementation detail when the v5 api-gateway prompt is actually written, not now.

### 2.4 Ownership enforcement on path-scoped endpoints

v2 introduced the JWT and the gateway's `X-User-Id` header, but never cross-checked it against the `{userId}` path variable used throughout order-service and recommendation-service — a gap named explicitly in v2's Known Limitations. v5 closes it.

**The problem, precisely:** a legitimately authenticated user (a valid, correctly-issued token, no forgery or privilege escalation involved) can act on another user's resources simply by changing the `{userId}` value in the URL, since nothing checks that the path id matches the token's own identity.

**Affected endpoints:**
- order-service: `GET /api/orders/cart/{userId}`, `POST /api/orders/cart/{userId}/items`, `DELETE /api/orders/cart/{userId}/items/{itemId}`, `POST /api/orders/checkout/{userId}`, `GET /api/orders/history/{userId}`
- recommendation-service: `GET /api/recommendations/user/{userId}`

**Mechanism:** one `HandlerInterceptor` per service (not per-endpoint logic) — `preHandle` extracts `{userId}` from the request path, compares it to the `X-User-Id` header forwarded by the gateway, and throws a new `ForbiddenException` on mismatch. Register it via a `WebMvcConfigurer`, applied only to the path patterns above — same registration pattern frontend-service's `LoginRequiredInterceptor` already uses (v1 Part B), just enforcing ownership instead of presence.

**Admin bypass (order-service only):** before rejecting a mismatch, the interceptor also reads `X-User-Role` (already forwarded by the gateway on every request per Section 2.5 — no gateway change needed for this). If `X-User-Role` equals exactly `"ADMIN"`, the mismatch check is skipped entirely and the request proceeds regardless of the `{userId}` path variable. This bypass exists specifically so the admin panel (Section 5) can view any user's order history through the same endpoint a customer uses for their own. This bypass applies to order-service's copy of the interceptor only — recommendation-service's copy is unmodified, since there is no legitimate admin use case for viewing another user's recommendations.

| Aspect | Behavior |
|---|---|
| Trigger | `X-User-Id` header value does not match the `{userId}` path variable **AND** (order-service only) `X-User-Role` is not exactly `"ADMIN"` |
| Bypass | order-service only: `X-User-Role: ADMIN` skips the mismatch check unconditionally, regardless of `{userId}` |
| Response | 403 Forbidden, `{"message": "You are not authorized to access this resource"}` — a new `ForbiddenException` wired into each service's existing `GlobalExceptionHandler`, same pattern as every other custom exception in the project |
| Not in scope | Role-based authorization (Section 2.5) is a separate concern — this section only fixes *ownership* (is this your own resource?), not *permission tiers* (what does your role allow you to do generally?). The admin bypass above is the one deliberate exception, added specifically to support the admin panel (Section 5). |

Does not touch product-service, user-service, payment-service, notification-service, api-gateway, or frontend-service — the gateway already forwards `X-User-Id` correctly (v2), and frontend-service already only ever sends the logged-in user's own id in the normal customer flow (the exploit path is hand-crafted requests via Postman/curl, not normal UI use). user-service's own admin endpoints are a separate mechanism, covered in Section 3.

**Extension — user-service's email-lookup endpoint (added in v4):** the same
`HandlerInterceptor` mechanism applies to `GET /api/users/{id}/email`, registered in
user-service alongside its existing `AdminOnlyInterceptor` (Section 3). One difference
from order-service/recommendation-service's copies: this endpoint has a second, legitimate
caller — order-service itself, calling directly via Eureka (bypassing the gateway entirely),
which never carries an `X-User-Id` header at all. So the check here is:
- `X-User-Id` present → must match `{id}` in the path, else 403 (blocks a customer from
  reading another user's email through the gateway)
- `X-User-Id` absent → allow through (this reliably means the request bypassed the
  gateway and was already vetted by user-service's own internal-secret filter, which runs
  before this interceptor in the filter/interceptor chain)
  No admin bypass is needed here, unlike order-service's copy — there's no admin-panel use
  case for looking up another user's email specifically (the admin panel already gets full
  user details, including email, via Section 3's `GET /api/users/{id}`).

### 2.5 Role-based authorization — admin-only product writes

Since v1, product-service's write endpoints (`POST /api/categories`, `POST /api/products`, `PUT /api/products/{id}`, `DELETE /api/products/{id}`, `PATCH /api/products/{id}/stock`) have been callable by any authenticated user — the `role` claim has existed on the JWT since v2 but nothing has ever read it. v5 closes this.

**The rule:** `role` must equal `"ADMIN"` for the five write endpoints listed above. `GET /api/products`, `GET /api/products/{id}`, and `GET /api/categories` remain open to any authenticated OR anonymous caller, unchanged from every prior version — this section restricts writes only, not reads.

**Mechanism — two pieces, one in the gateway, one in product-service:**

1. **api-gateway** (`JwtValidationFilter`, extended): when forwarding a request, in addition to `X-User-Id` and `X-User-Email`, also forward the JWT's `role` claim as a new header, `X-User-Role`. This is additive to the existing header-forwarding logic from v2 — no change to whitelist behavior, no change to which requests require a token at all. This header is what makes both Section 2.4's admin bypass and Section 3's admin-only endpoints possible — it is forwarded on every request, not just product-service writes.
2. **product-service** (new `HandlerInterceptor`, `AdminOnlyInterceptor`): registered via `WebMvcConfigurer`, applied only to the five write-endpoint path/method combinations listed above (not the GET endpoints). `preHandle` reads `X-User-Role`; if it is missing or not exactly `"ADMIN"`, throw a new `ForbiddenException` → 403, `{"message": "This action requires ADMIN privileges"}`. Same `GlobalExceptionHandler` wiring pattern as everywhere else in the project. This is a new, separate `ForbiddenException` class local to product-service — Section 2.4's `ForbiddenException` lives in order-service and recommendation-service only, not shared code. Section 3's `ForbiddenException` (user-service) is likewise its own separate local class.

**How a user actually becomes ADMIN in this project:** there is still no endpoint anywhere to change a user's role after registration (`role` defaults to `"CUSTOMER"` in user-service's `@PrePersist`, per v1, and nothing in v2–v4 ever offered a way to change it). For v5's purposes, promoting a test user to ADMIN is a **manual database update** — directly `UPDATE users SET role = 'ADMIN' WHERE id = ...` in Postgres, done by hand for testing. Building a proper admin-promotion flow (another admin promoting a user, or a seed/bootstrap mechanism) is explicitly out of scope for this version — name it as a known limitation, not an oversight.

**What this does not cover:** no role check is added to order-service, recommendation-service, payment-service, or notification-service — none of their endpoints are role-differentiated in this project's design (e.g. any authenticated CUSTOMER can check out their own cart, view their own history). Role enforcement is scoped to product-service's writes (this section) and user-service's admin endpoints (Section 3) only — those are the only two places in the whole system where an admin/customer distinction is meaningful per the project's design.

### 2.6 Admin-only restriction on `containing-product`

**The problem this closes:** `GET /api/orders/containing-product/{productId}` has existed since v3, but it was only ever called one way — service-to-service, recommendation-service → order-service, direct via Eureka, authenticated by the internal-secret filter. As of v5, the gateway's existing `order-route` (`Path=/api/orders/**`, unchanged since v2) already matches this path, and `JwtValidationFilter`'s whitelist is an allow-list, not a deny-list — so any authenticated CUSTOMER token, not just ADMIN, could already call it through the gateway and see every userId that bought a given product. This section closes that gap before Section 5.4a relies on the endpoint for the admin panel.

**The rule:** when reached with `X-User-Id` present (i.e. via the gateway), `X-User-Role` must equal exactly `"ADMIN"`. When reached with no `X-User-Id` at all (recommendation-service's existing direct-via-Eureka call, internal-secret-authenticated, unchanged since v3), allow through unconditionally — same reasoning Section 2.4 already used for user-service's `/email` endpoint.

| Aspect | Behavior |
|---|---|
| Trigger | `X-User-Id` present **AND** `X-User-Role` missing or not exactly `"ADMIN"` |
| Bypass | `X-User-Id` absent entirely → allow through unconditionally (recommendation-service's v3 internal call) |
| Response | 403 Forbidden, same shape as every other `ForbiddenException` in this project |
| Not in scope | Cart/checkout/history endpoints are governed by Section 2.4 only, unchanged by this section |

**Mechanism:** reuse order-service's existing `ForbiddenException` and `GlobalExceptionHandler` mapping from Section 2.4 — do not create a second exception class. Add one more `if` branch inside the **same** `HandlerInterceptor` Section 2.4 already registers: when the path matches `/api/orders/containing-product/**`, apply the rule above instead of the `{userId}`-vs-`X-User-Id` comparison (this path has no `{userId}` variable to compare against). Register this path pattern alongside the five Section 2.4 already lists, in the same `WebMvcConfigurer` call — one interceptor per service, not a second one.

---

---

## 3. Admin endpoints — user-service

Three new endpoints, all admin-only, supporting the admin panel's user-management pages (Section 5.5).

| Method | Path | Purpose | Notes |
|---|---|---|---|
| GET | `/api/users` | list all users | returns `List<UserResponse>` — the existing DTO from v1, already excludes `passwordHash`, reused as-is, no new DTO |
| GET | `/api/users/{id}` | get one user by id | returns `UserResponse`; throw a new `UserNotFoundException` → 404 if the id doesn't exist; added for consistency with the REST shape every other single-resource lookup in this project already uses, and because the admin user-detail page (Section 5.5) needs to resolve one user by id rather than filtering the full list client-side |
| DELETE | `/api/users/{id}` | delete a user | hard delete, 204 on success, same `UserNotFoundException` → 404 if the id doesn't exist |

**Mechanism:** new `AdminOnlyInterceptor` in user-service (package `com.ecommerce.userservice.security`), registered via `WebMvcConfigurer`, applied only to these three path/method combinations — not `/api/users/register`, `/api/users/login`, or `/api/users/me`, which are unaffected and unchanged. `preHandle` reads `X-User-Role` (forwarded by the gateway per Section 2.5); if missing or not exactly `"ADMIN"`, throw a new `ForbiddenException` (local to user-service, not shared with product-service's or order-service's copies) → 403, `{"message": "This action requires ADMIN privileges"}`. Same `GlobalExceptionHandler` wiring pattern as every other custom exception in the project.

**Known gap, stated explicitly, not solved here:** deleting a user does not cascade to or clean up their data in order-service. Orders reference `userId` as a plain `Long` (never a JPA foreign key, by design since v1 — see `architecture-v1.md` Section 5.1), so a deleted user's past orders simply remain in order-service, referencing an id that no longer resolves to a real user. A real system would soft-delete users or reassign/anonymize their orders; this project accepts the gap rather than solving it, consistent with how cross-service referential integrity has been handled everywhere else in the project (plain reference ids, no cross-schema FK, since these are separate databases entirely).

---

## 4. Stock visibility — customers vs admins

**The rule:** product-service's API is unchanged by this section — `GET /api/products` and `GET /api/products/{id}` keep returning the real `stockQuantity` integer exactly as always, since v1. This is a **frontend-only display decision**, not a backend contract change — no new endpoint, no new DTO field, no product-service code touched.

- **Customer-facing pages (`product-detail.html`, `home.html`, and any other page showing a product card):** replace the numeric stock display with a derived boolean — "In Stock" if `stockQuantity > 0`, "Out of Stock" otherwise. The actual number is never rendered anywhere a customer can see it.
- **Admin product-list page (Section 5.4):** shows the real `stockQuantity` number, since managing inventory requires it.

**Mechanism:** frontend-service's `GlobalModelAttributes` (existing class from v1 Part B) gains one more `@ModelAttribute` method, `loggedInUserRole`, reading `HttpSession` attribute `"userRole"` (new session attribute — see Section 5.1), returning it or null if absent. Templates use `th:if="${loggedInUserRole == 'ADMIN'}"` to decide which stock view to render, same conditional pattern the header fragment already uses for `loggedInUserName`.

---

## 5. Admin panel — frontend-service

A dedicated `/admin/**` section of frontend-service, reachable only to users with the `ADMIN` role, covering product/category/stock management and user management. No new service — this is entirely new controllers, templates, and Feign client extensions inside the existing frontend-service project.

### 5.1 Session change

`AuthController`'s login flow (v1 Part B) currently stores `sessionToken`, `userId`, `userName` in `HttpSession`. Add one more attribute: `userRole`, taken from the same `getCurrentUser` call already made immediately after login (it returns `UserResponse`, which has always included `role` — no new Feign call needed, just capture a field that was already being fetched and previously discarded).

### 5.2 New interceptor: AdminRequiredInterceptor

Package `com.ecommerce.frontendservice.config`. Same shape as the existing `LoginRequiredInterceptor` (v1 Part B): `preHandle` checks `HttpSession` attribute `"userRole"`. If not present or not exactly `"ADMIN"`, redirect to `/` and return `false` — a non-admin is silently bounced home, not shown a 403/error page, so the admin panel's existence isn't advertised to a probing customer. Registered via the existing `WebMvcConfigurer`, applied to path pattern `/admin/**`.

This interceptor is layered **on top of** `LoginRequiredInterceptor`, not a replacement for it — `/admin/**` needs both active (must be logged in at all, and must be an admin specifically), the same way Spring MVC allows multiple interceptors on overlapping path patterns.

**Security note:** this interceptor is a UX convenience only, not the actual security boundary — the real enforcement is Section 2.5's `AdminOnlyInterceptor` on product-service and Section 3's `AdminOnlyInterceptor` on user-service, both of which check `X-User-Role` independently of anything the frontend does. Even if this frontend interceptor were somehow bypassed, the backend calls it fronts would still reject a non-admin.

### 5.3 New Feign client methods

Extend the existing `ProductServiceClient` (targeting `"api-gateway"`, same pattern as every client since v2) with the write methods it never needed before:
- `POST /api/products`, `PUT /api/products/{id}`, `DELETE /api/products/{id}`, `PUT /api/products/{id}/stock`
- `POST /api/categories`, `GET /api/categories` (category list needed to populate the product-creation form's dropdown)

Extend `UserServiceClient` with:
- `GET /api/users`, `GET /api/users/{id}`, `DELETE /api/users/{id}`

All of these ride on the existing `FeignAuthInterceptor` (v2) — the JWT is attached automatically to every call made by a client targeting `"api-gateway"`, no new auth code needed on the frontend side.

### 5.4 New controller: AdminProductController

Package `com.ecommerce.frontendservice.controller`, base path `/admin`.

| Route | Calls | Purpose |
|---|---|---|
| `GET /admin/products` | `GET /api/products` (existing paginated endpoint) | list all products, showing real stock counts (Section 4), with edit/delete/adjust-stock actions per row, plus a "View buyers" link per row (Section 5.4a) |
| `GET /admin/products/new` | `GET /api/categories` | blank product creation form, category dropdown populated from the category list |
| `POST /admin/products/new` | `POST /api/products` | create; on a 400 from product-service (validation failure), redisplay the form with the error rather than a generic error page |
| `GET /admin/products/{id}/edit` | `GET /api/products/{id}` | prefilled edit form |
| `POST /admin/products/{id}/edit` | `PUT /api/products/{id}` | update, redirect to `/admin/products` |
| `POST /admin/products/{id}/delete` | `DELETE /api/products/{id}` | delete, redirect to `/admin/products` |
| `POST /admin/products/{id}/stock` | `PUT /api/products/{id}/stock` | small inline form, single `delta` field, redirect to `/admin/products` |
| `GET /admin/categories/new` | — | blank category creation form |
| `POST /admin/categories/new` | `POST /api/categories` | create; on a 409 (duplicate name), redisplay the form with the error |

### 5.4a Admin: view buyers of a product

Read-only. No new Feign client interface — extends two that already exist.

Extend `OrderServiceClient` (frontend-service, `.client`) with: `@GetMapping("/api/orders/containing-product/{productId}") List<OrderResponse> getOrdersContainingProduct(@PathVariable("productId") Long productId)` — declarative, no body. `OrderResponse` is the existing DTO, already carries `userId` per order — no new DTO. Rides on the existing `FeignAuthInterceptor` (v2) automatically, which is what makes Section 2.6's role check upstream see the admin's role.

New route on `AdminProductController`:

| Route | Calls | Purpose |
|---|---|---|
| `GET /admin/products/{id}/buyers` | `GET /api/products/{id}` (existing) + `orderServiceClient.getOrdersContainingProduct(id)` (new) | shows every PLACED order containing this product |

Logic: fetch the product (404 via existing `ProductNotFoundException` if missing) → fetch orders containing it → extract distinct `userId`s (dedup) plus total order count (not deduped — show both numbers) → for each distinct `userId`, call `userServiceClient.getUserById(userId)` (already added in Section 5.3) to resolve name/email, wrapped in try/catch, skipping a row on individual failure rather than failing the whole page (guards against Section 3's known user-deletion gap) → Model gets `product`, `totalOrders`, `buyers` (reusing `UserResponse`, no new DTO) → view name `admin-product-buyers`.

New template `admin-product-buyers.html`: header fragment, product name + "Purchased N times by M distinct users", a plain table of buyer name/email each linking to the existing `/admin/users/{id}` page, no JavaScript.

Not in scope for this addition: no date filtering, no CSV export, no cross-product "top buyers" aggregate view, no change to `containing-product`'s response shape.

---

### 5.5 New controller: AdminUserController

Package `com.ecommerce.frontendservice.controller`, base path `/admin`.

| Route | Calls | Purpose |
|---|---|---|
| `GET /admin/users` | `GET /api/users` | list all users: id, name, email, role |
| `GET /admin/users/{id}` | `GET /api/users/{id}` for the user's own details, plus `GET /api/orders/history/{id}` for their order history (existing order-service endpoint — reachable for any `{userId}` here specifically because of Section 2.4's admin bypass) | user detail page combining profile info and order history in one view |
| `POST /admin/users/{id}/delete` | `DELETE /api/users/{id}` | delete, redirect to `/admin/users` |

### 5.6 New templates

`admin-products.html`, `admin-product-form.html` (shared by new/edit), `admin-category-form.html`, `admin-users.html`, `admin-user-detail.html`, `admin-product-buyers.html` (Section 5.4a). Same plain-HTML-forms, no-JavaScript constraint as every other template in this project — no exceptions for the admin panel. Header fragment (existing, from v1 Part B) gains one more conditional link — "Admin Panel" → `/admin/products`, shown only `th:if="${loggedInUserRole == 'ADMIN'}"`, alongside the existing logged-in-user links.

### 5.7 What NOT to build in this version

- No bulk actions (bulk delete, bulk stock adjust) — one row at a time only.
- No admin-promotion flow — Section 2.5 already established this is manual-SQL-only; this section doesn't change that.
- No pagination UI on `/admin/users` — a plain full list is acceptable at this project's scale, same reasoning v3 gave for recommendation-service's lack of caching.
- No delete-confirmation modal (that would need JavaScript) — if you want a safety step before a destructive action, use a second page/form (e.g. `GET /admin/products/{id}/delete-confirm` rendering a plain "are you sure?" page with a POST button) rather than a JS confirm dialog. Optional, not required for this version to be "done."
- No audit log of admin actions (who changed what, when) — not tracked anywhere in this project's design.

---

## 6. Standardized error response shape

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

Same `@RestControllerAdvice` pattern each service already has individually since v1 — v5 doesn't add new exception *types* beyond what Sections 2.4, 2.5, and 3 introduce, it makes all six backend services agree on the same response *shape* so a client (or you, in a viva) can rely on error responses looking identical regardless of which service produced them.

---

## 7. Swagger / OpenAPI

Add `springdoc-openapi-starter-webmvc-ui` to product-service, user-service, order-service, recommendation-service, payment-service, notification-service. Each gets its own `/swagger-ui/index.html`. No custom configuration needed beyond the dependency itself and, optionally, a one-line `@OpenAPIDefinition` title/description per service.

---

## 8. Targeted tests (not full coverage)

Pick tests that prove something specific, not tests for coverage percentage's sake.

| Service | Test | What it proves |
|---|---|---|
| order-service | Checkout with insufficient stock on item 2 of a 3-item cart | v2's "validate all items before decrementing any" logic actually holds |
| order-service | Checkout where payment-service returns FAILED | v4's restock-on-failure logic correctly reverses the earlier decrement |
| order-service | Request to `/api/orders/checkout/{userId}` where `{userId}` doesn't match `X-User-Id`, and `X-User-Role` is `CUSTOMER` or absent | Section 2.4's ownership interceptor returns 403 and never reaches the checkout logic |
| order-service | Same mismatched-`{userId}` request, but with `X-User-Role: ADMIN` | Section 2.4's admin bypass correctly allows the request through despite the mismatch |
| recommendation-service | A known small dataset run through the v4 scoring formula | the score isn't hand-waved — assert the exact expected ranking against a formula you can also compute by hand |
| product-service | `POST /api/products` with `X-User-Role: CUSTOMER` | Section 2.5's admin-only interceptor returns 403 and never reaches the controller |
| product-service | `POST /api/products` with `X-User-Role: ADMIN` | the same interceptor correctly allows the request through when the role is right |
| user-service | `GET /api/users` with `X-User-Role: CUSTOMER` | Section 3's admin-only interceptor returns 403 and never reaches the controller |
| user-service | `GET /api/users` with `X-User-Role: ADMIN` | the same interceptor correctly allows the request through |
| api-gateway | An authenticated-route request with no `Authorization` header | returns 401 and never reaches the downstream service |

Tests like these, each provably tied to a specific design decision from an earlier version file (or this one), are worth more in a viva than broad but shallow coverage across everything.

---

## 9. Known-limitations writeup — out of scope for this file

The consolidated known-limitations writeup (pulling together every "Known limitations" section from `architecture-v1.md` through this file) is tracked in `architecture-master-methodology.md`, Section 11 — a human-only file, never pasted into Copilot/Amazon Q. This section is intentionally left as a pointer only, so nothing in this AI-facing file is addressed to a human reader rather than the model being prompted with it.

Items worth flagging for that writeup specifically, since this version resolves some and introduces others:
- The `userId`-vs-JWT ownership gap named in `architecture-v2.md` is closed as of Section 2.4 above — do not carry it forward as unresolved. Note its one deliberate exception: the admin bypass on order-service, added to support Section 5's admin panel — this is an intentional design choice, not a re-opened version of the original gap (it's role-gated, not a blanket bypass).
- Role-based authorization is now enforced for product-service writes (Section 2.5) and user-service's admin endpoints (Section 3), but the *mechanism to grant the ADMIN role itself* remains manual (direct database edit, no promotion endpoint) — this residual gap should still be named.
- Deleting a user via the admin panel does not cascade to or clean up their data in order-service (Section 3) — their past orders remain, referencing a `userId` that no longer resolves. Named explicitly as an accepted gap, not an oversight.

---

## 10. Testing checklist before calling v5 — and the whole project — "done"

- [ ] Manually stop payment-service, attempt a checkout, confirm order lands in `PAYMENT_PENDING_RETRY` (not `PAYMENT_FAILED`, not a raw 500)
- [ ] Manually stop order-service or product-service, hit a recommendation endpoint, confirm it returns an empty list, not an error
- [ ] Every backend service's error responses match the shape in Section 6 exactly, for at least one triggered error per service
- [ ] All six backend services' Swagger UIs load and show accurate endpoint lists
- [ ] Log in as user A, obtain their JWT, attempt `GET /api/orders/cart/{userB's id}` using user A's (CUSTOMER-role) token — confirm 403, not 200
- [ ] Same check repeated for checkout, history, and recommendation-service's user endpoint — confirm all five affected endpoints reject a mismatched userId with 403 for a CUSTOMER-role token
- [ ] Confirm a matching userId (own resource) still returns 200 as before — the ownership interceptor must not break the normal, correct-owner case
- [ ] Manually promote a test user to ADMIN via direct SQL update; confirm that user's token CAN successfully call `GET /api/orders/history/{someOtherUserId}` on order-service (Section 2.4's admin bypass), while a CUSTOMER-role token on the same request still gets 403
- [ ] Confirm that user CAN create/update/delete products and adjust stock on product-service
- [ ] Confirm a CUSTOMER-role user gets 403 attempting the same five product-service write endpoints
- [ ] Confirm GET endpoints on product-service still work for both roles, and for anonymous/no-token requests, unaffected by Section 2.5
- [ ] Confirm `GET /api/users`, `GET /api/users/{id}`, and `DELETE /api/users/{id}` on user-service all return 403 for a CUSTOMER-role token and succeed for an ADMIN-role token
- [ ] Customer-facing product pages show only "In Stock"/"Out of Stock," never a number, for both logged-out visitors and CUSTOMER-role logged-in users
- [ ] Admin's `/admin/products` page shows the real numeric stock count
- [ ] A CUSTOMER-role user navigating directly to any `/admin/**` URL in the browser is redirected home, not shown admin content or an error page
- [ ] Full admin walkthrough: log in as ADMIN → create a category → create a product → adjust its stock → edit it → view it as a logged-out/customer visitor and confirm the changes reflect (and stock still shows only In/Out of Stock) → delete it → confirm it's gone from customer-facing pages
- [ ] Full user-management walkthrough: as ADMIN, view the user list → click into one user → see their real order history on the detail page → delete a disposable test user → confirm they're gone from the list and can no longer log in
- [ ] Log in as a CUSTOMER-role user, obtain their JWT, attempt `GET /api/orders/containing-product/{anyProductId}` directly through the gateway — confirm 403, not 200 (Section 2.6)
- [ ] Confirm recommendation-service's existing internal call to the same endpoint (direct via Eureka, no `X-User-Id`) still succeeds unchanged — re-run one of v3's original recommendation checklist items and confirm it still passes after this change
- [ ] Full walkthrough: as ADMIN, open `/admin/products`, click "View buyers" on a product with at least two known purchasers (including one purchased twice by the same user) — confirm the correct distinct-buyer count, the correct total-orders count, and that clicking a listed buyer navigates to their existing Section 5.5 user-detail page
- [ ] With a test user deleted via the admin panel (Section 3) after having purchased the product being viewed, confirm the buyers page still loads (that row is silently skipped, not a 500)
- [ ] All tests in Section 8 pass
- [ ] The known-limitations writeup exists in `architecture-master-methodology.md` Section 11 and covers every item listed there, including the v5-specific notes in Section 9 above
- [ ] Full end-to-end walkthrough (register → browse → view products with stock hidden → cart → checkout with a real payment success → checkout with a forced payment failure → recommendations reflect purchase/view history → admin-only product write correctly blocked for a customer, correctly allowed for an admin → admin panel product/category/stock management works → admin panel user list/detail/delete works) works with zero unhandled errors anywhere in the chain