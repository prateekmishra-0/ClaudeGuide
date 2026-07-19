# ARCHITECTURE — v4 (Recommendation v2 + Payment + Notifications)

> Carried over from v3: eureka-server, api-gateway, product-service, user-service, order-service, recommendation-service, frontend-service all exist — see `services-index.md` for the summary. This file covers what's new or changed in v4: view tracking + weighted scoring in recommendation-service, a new payment-service, a new notification-service (now sending real email, see Section 7), a checkout-flow change in order-service, a new email-lookup endpoint on user-service, and a frontend card-entry form for realism.
> This is the busiest version — expect it to take the largest share of your remaining credits.
> **Revision note:** this version was revised before any v4 prompt was written, to add two things beyond the original scope: (1) a frontend card-entry form with real server-side validation, purely cosmetic — its data is never forwarded to payment-service, and (2) real email delivery in notification-service via Gmail SMTP, instead of a console log line. Registration/payment OTP verification was considered and explicitly deferred to v6 — it changes User's tested schema and login behavior, which is version-shaped work, not a v4 patch.

---

## 1. Services in this version

| Service | Port | Change from v3 |
|---|---|---|
| eureka-server | 8761 | unchanged |
| api-gateway | 8000 | two new routes (Section 5) |
| product-service | 8081 | **new entity**: ProductView (Section 2) |
| user-service | 8082 | **new endpoint**: internal email lookup by id (Section 6a) |
| order-service | 8083 | checkout flow expanded to 3 states, calls payment-service and (new) user-service, then notification-service (Section 6) |
| recommendation-service | 8084 | scoring logic upgraded, calls the new view-tracking data (Section 3) |
| payment-service | 8085 | **new** — gains the internal-secret filter (Section 6.1); never receives card details, only `{orderId, amount}` |
| notification-service | 8086 | **new** — gains the internal-secret filter (Section 6.1); sends a real email via Gmail SMTP (Section 7) |
| frontend-service | 8080 | view-tracking call added; checkout gains a card-entry form (Section 8); checkout confirmation page reflects payment result |

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

4 fields. **No card fields anywhere on this entity or its request body** — see Section 4.4.

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

### 4.4 Explicit boundary: card details never reach this service

The frontend's card-entry form (Section 8) collects a card number, CVV, and expiry for UI realism only. **None of those fields are part of this endpoint's request body, are transmitted to payment-service, or are persisted anywhere.** The `Payment` entity and the `POST /api/payments` contract are unchanged from the original v4 design — `{orderId, amount}` in, `{status, paymentId}` out. This boundary is deliberate: this project never handles real payment credentials, and nothing about the card form should imply otherwise. State this explicitly in the payment-service prompt so a fresh chat doesn't "helpfully" add card fields to the entity or endpoint.

---

## 5. api-gateway changes

| Route ID | Predicate | Destination |
|---|---|---|
| payment-route | `Path=/api/payments/**` | `lb://payment-service` |
| notification-route | `Path=/api/notifications/**` | `lb://notification-service` — only needed if you expose a manual trigger/lookup; if notification-service is purely called service-to-service from order-service with no external route, this entry is optional |

Both entries (if notification-route is included at all) are appended to the same `spring.cloud.gateway.server.webmvc.routes` list from v2/v3 — not a new property block. See promptrule.md's route-configuration rule.

Both require a valid JWT, same as everything since v2.

---

## 6a. user-service changes — new internal email-lookup endpoint

This is new in this revision. Order-service (Section 6) needs a way to resolve a user's email address from their `userId` so notification-service has somewhere real to send to. user-service already stores `email` on `User` (since v1) — no entity change, just one new read endpoint.

### 6a.1 New endpoint

| Method | Path | Purpose | Notes |
|---|---|---|---|
| GET | `/api/users/{id}/email` | returns `{"email": "..."}` for the given user id | internal, service-to-service only — see 6a.2 |

### 6a.2 Access boundary — deliberately internal, not gateway-fronted for external use

This endpoint is called directly by order-service via Feign, resolved through Eureka by service name (`@FeignClient(name = "user-service")`) — the same direct-call pattern order-service already uses for product-service since v1. It never needs to go through api-gateway for this purpose.

It is still protected by user-service's existing `InternalSecretFilter` from v2 (no whitelist, applies to every endpoint), so a direct hit without `X-Internal-Secret` gets a 401 regardless of caller.

**Known gap, stated explicitly rather than silently left implicit:** because api-gateway's existing `user-route` predicate (`Path=/api/users/**`) already matches this new path, any authenticated user (any valid JWT) could also reach `GET /api/users/{id}/email` *through* the gateway and look up another user's email address — the gateway's JWT filter has no per-user ownership check on this path (that class of gap is the same one v2 named for order-service/recommendation-service paths, and v5 only closes it for the specific endpoints listed in that file — this new one isn't in that list). Accept this as a known limitation for v4, name it in your writeup, and optionally extend v5's ownership-interceptor list to cover it if you want it closed before the demo. Not fixing it now is a legitimate call for a localhost demo project, same reasoning already applied to the other v2-era gaps.

---

## 6. order-service changes — checkout flow, now 3 states

### 6.1 Internal-secret filter — payment-service, notification-service, user-service (new caller), and order-service's new outbound calls

Same mechanism as `architecture-v2.md` Section 2.3 and `architecture-v3.md` Section 2.6, extended to this version's new callees:

- **payment-service and notification-service (checkers):** both get the same `OncePerRequestFilter`, `@Order(1)`, checking `X-Internal-Secret` against `internal.secret` in each service's own `application.yml` — no whitelist, applies to every endpoint including `POST /api/payments` and `POST /api/notifications`.
- **order-service (sender, expanded role):** order-service was already attaching `X-Internal-Secret` on its calls to product-service since v2. This version adds three more outbound calls that need the same header attached: `POST /api/payments` (step 3 below), `GET /api/users/{id}/email` (new step 3.5 below), and `POST /api/notifications` (steps 4a/4b). All four of order-service's outbound Feign targets — product-service, payment-service, user-service, notification-service — now require this header; none of this traffic passes through api-gateway.

**Configuration:** payment-service's and notification-service's `application.yml` each need `internal.secret` set to the same literal string already shared across every other service since v2 — no new value generated for this version. This brings the total to seven services holding this string: api-gateway, product-service, user-service, order-service, recommendation-service, payment-service, notification-service.

As with recommendation-service in v3, a missing or wrong secret on order-service's calls to any of these would surface as a rejected request at the receiving service's filter — worth checking directly if checkout behavior looks wrong (e.g. orders stuck, no email arriving) before assuming the business logic itself is broken.

`Order.status` gains three new valid values: `PENDING_PAYMENT`, `PAID`, `PAYMENT_FAILED` (in addition to v1's `CART`, `PLACED` — note `PLACED` is effectively superseded by `PAID`/`PAYMENT_FAILED` going forward; keep `PLACED` in the codebase for backward compatibility with any v1/v2/v3 test data, but new checkouts never land there again).
The Order entity's existing @PrePersist/@PreUpdate lifecycle hooks (v1) contain a hardcoded validation check restricting status to "CART"/"PLACED" only — this must be widened to also accept "PENDING_PAYMENT", "PAID", and "PAYMENT_FAILED", or every v4 checkout will fail with a 500 the instant the order is saved with a new status value. This is a required entity-level change in this version, not an unrelated v1 detail — do not treat Order as unchanged when building this version's order-service prompt.

```
POST /api/orders/checkout/{userId}
  1. (unchanged from v2) validate stock for all items, decrement stock for all items
  2. Create the order with status PENDING_PAYMENT
  3. Call payment-service: POST /api/payments {orderId, amount: order total}
     — the card fields collected on the frontend (Section 8) are NOT part of this call's body;
       payment-service still only ever sees {orderId, amount}
  3.5. Call user-service: GET /api/users/{userId}/email — resolve the email to notify.
       Wrap this call in try/catch: if it fails, proceed with the status transition below
       regardless (do not fail checkout just because the email lookup failed), and skip the
       notification-service call entirely in that case rather than sending a notification with
       no recipient.
  4a. SUCCESS → order status → PAID → if an email was resolved in 3.5, call notification-service
              (Section 7) with {orderId, email, message: success text}
  4b. FAILED  → order status → PAYMENT_FAILED
              → for EVERY item in the order, call product-service PUT /api/products/{id}/stock
                with a POSITIVE delta equal to the quantity (restock what step 1 decremented)
              → if an email was resolved in 3.5, call notification-service with
                {orderId, email, message: failure text}
  5. Return the order with its final status
```

The restock-on-failure step in 4b is the one piece of real compensation logic in v4 — it's a simplified, synchronous version of what a saga pattern would do properly in a real distributed system with async messaging. Worth stating exactly that in your prompt to Copilot, and worth stating exactly that in your own "known limitations" writeup too — it's good, honest engineering framing.

The email-lookup-then-notify sequence (3.5, 4a, 4b) is new in this revision and is the one part of the checkout flow with an explicit fail-open branch — a missing email or a down notification-service must never block or fail an otherwise-successful checkout.

---

## 7. notification-service (new) — real email via Gmail SMTP

No entity, no database. A single endpoint called synchronously by order-service, not event-driven (no message broker available given the no-Docker constraint — this is a stated tradeoff, see Known Limitations).

### 7.1 New dependency

Since this project hasn't started v4 yet, generate notification-service's Spring Initializr project with **Spring Web, Eureka Discovery Client, Spring Boot Starter Mail, Lombok** — `spring-boot-starter-mail` is required from the start this time (not added mid-stream), so no promptrule violation.

### 7.2 Endpoint

| Method | Path | Purpose |
|---|---|---|
POST /api/notifications — body: {orderId, email, message} — sends a real email via Gmail SMTP (spring-boot-starter-mail, JavaMailSender), and separately logs the outcome (INFO on success or attempt, ERROR on failure) so a local record always exists even if Gmail is unreachable. The DTO also includes an email field, not present in the original baseline spec, since a real send needs a destination address. This project moved past the "optional polish" framing originally described here — real email is the actual built behavior, not console-log-only.

Note the payload gained `email` compared to the original v4 design (which was just `{orderId, message}`) — this is the one contract change needed to make real sending possible, supplied by order-service's new Section 6.1's step 3.5.

### 7.3 Mail configuration

`application.yml`:
```yaml
spring:
  mail:
    host: smtp.gmail.com
    port: 587
    username: <your Gmail address>
    password: <a Gmail App Password — NOT your real Google account password>
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
```

An **App Password** is a 16-character code Google generates for exactly this purpose (Google Account → Security → 2-Step Verification → App Passwords) — it's scoped to SMTP use and revocable independently of your real password, which is why it's used here instead of your actual account credentials. Gmail's free tier allows roughly 500 sends/day on a regular account, comfortably enough for a demo.

Since you've said this codebase stays on your machine and gets deleted after the demo, hardcoding the App Password directly in `application.yml` (rather than pulling it from an environment variable) is a reasonable call for this project's actual threat model — just don't push this file to a public GitHub repo before deleting it, since that's the one way it'd leave your machine unintentionally.

### 7.4 Sending logic

Use `JavaMailSender` (auto-configured once `spring-boot-starter-mail` is on the classpath and the properties above are set) to send a plain-text email: `to` = the `email` field from the request body, `subject` = something fixed like "Order Update" or derived from the message, `text` = the `message` field. Wrap the actual `send()` call in try/catch — if the send fails (bad credentials, network issue, Gmail rate limit), log the failure but still return 201 from the endpoint; a failed notification email must never surface as an error back to order-service, consistent with the fail-open principle already established for recommendation-service in v3.

If you have spare time near the end of the project, adding an HTML template for the email body is optional polish — not required for v4 to be "done."

---

## 8. frontend-service changes

### 8.1 View tracking (unchanged from original v4 design)

- Product detail page fires `POST /api/products/{id}/view` on page load, for logged-in users only, fire-and-forget (don't block rendering on this call's response)

### 8.2 Card-entry form on checkout — new in this revision

Added to the checkout flow, before the existing checkout POST is triggered: a form collecting card number, CVV, and expiry, for realism only.

**Fields and validation (server-side, no JavaScript — this project has been JS-free since v1):**

| Field | Rule |
|---|---|
| Card number | exactly 16 digits, numeric only — `@Pattern(regexp = "^\\d{16}$")` on the form-backing DTO |
| CVV | exactly 3 digits — `@Pattern(regexp = "^\\d{3}$")` |
| Expiry | `MM/YY` format — `@Pattern(regexp = "^(0[1-9]|1[0-2])/\\d{2}$")`, plus a small custom check (or a `@AssertTrue`-annotated method on the DTO) confirming the MM/YY isn't already in the past relative to the current date |

Use the same `@Valid` + `BindingResult` pattern already established elsewhere in this project's controllers (e.g. registration/login form validation from Part B) — on a validation failure, re-render the checkout page with field-level error messages, do not proceed to call order-service's checkout endpoint.

**This form's data goes nowhere beyond this validation step.** Once validation passes, the existing `POST /api/checkout` flow to order-service proceeds exactly as it already does — `{}`/no body, or whatever the existing call shape already is — the card number, CVV, and expiry are not attached to that request, not stored in the session, not logged. They exist purely so the checkout page feels like a real payment step; the actual "processing" is still payment-service's deterministic mock from Section 4, decided entirely by the order's total amount.

Confirm this boundary explicitly in the frontend-service prompt for this section, so a fresh chat doesn't infer that the card fields should be threaded through to the backend.

### 8.3 Checkout confirmation page

Checkout confirmation page now shows the actual payment outcome (`PAID` vs `PAYMENT_FAILED`) instead of the old generic "order placed" message from v1/v2/v3.

---

## 9. Known limitations (intentional, fixed in later versions)

- notification-service is called synchronously via direct REST, not through an event bus — a real message broker (Kafka/RabbitMQ) is the textbook-correct choice, ruled out here because Docker isn't available; this is a known, deliberate trade-off, not an oversight
- Restock-on-payment-failure is the only compensation logic in the system — a crash between steps in the checkout flow (e.g. server dies after decrementing stock but before calling payment-service) leaves the order stuck in an inconsistent state with no automatic recovery; acceptable to leave unsolved and name explicitly in your final writeup
- No retry on a failed payment call itself (a payment-service timeout is treated the same as a payment-service rejection) — v5's circuit breaker section addresses this distinction
- Payment mock logic is deterministic but arbitrary (the `.13` cents rule) — fine for testing, obviously not realistic
- The card-entry form (Section 8.2) is cosmetic only — no real card network validation (Luhn check, issuer BIN lookup, etc.) is performed, and no real charge is ever attempted. Worth naming explicitly so it's clear this was a deliberate scope choice, not a security gap in a system that was supposed to process real payments.
- `GET /api/users/{id}/email` (Section 6a) is reachable through api-gateway by any authenticated user, not just order-service, since it falls under the existing `/api/users/**` route with no per-endpoint ownership check — a real (if narrow) information-disclosure gap, same category as the userId-vs-JWT gap named in v2, not closed until/unless you extend v5's ownership work to cover it explicitly.
- Same internal-secret caveat as v2/v3 — now seven services hold the identical static string. No new mitigation in v4; still an accepted trade-off for a localhost demo project.
- Registration/payment OTP verification (email-code confirmation before an account or payment is considered valid) was scoped out of v4 entirely and deferred to v6, since it changes user-service's tested schema and login behavior — that's dedicated-version work, not a v4 addition.

---

## 10. Testing checklist before calling v4 "done"

- [ ] `POST /api/products/{id}/view` records rows; `GET /api/products/{id}/views/count-by-viewer` returns correct counts
- [ ] Recommendation scores visibly change when you manually generate extra views for a product (compare before/after)
- [ ] Checkout page rejects a card number that isn't exactly 16 digits, a CVV that isn't exactly 3 digits, and an expiry that's already past — with field-level error messages, no crash, no progression to the checkout call
- [ ] Checkout page accepts a valid-looking card (16 digits, 3-digit CVV, future MM/YY) and proceeds to call order-service normally
- [ ] Confirm (via logging or a debugger breakpoint) that the card number/CVV/expiry values are never present in the request body order-service or payment-service actually receive
- [ ] A payment amount that triggers the `.13` rule returns FAILED, and the corresponding order ends up in `PAYMENT_FAILED` with stock correctly restocked
- [ ] A normal payment amount returns SUCCESS, order ends up `PAID`, stock stays decremented (not restocked)
- [ ] `GET /api/users/{id}/email` returns the correct email for a known user id, and 401s with no `X-Internal-Secret` header
- [ ] Complete a real checkout (success path) and confirm a real email actually arrives in the test account's inbox, not just a console log line
- [ ] Complete a checkout that triggers `PAYMENT_FAILED` and confirm a distinct failure-message email arrives
- [ ] Temporarily use a wrong Gmail App Password, attempt a checkout, and confirm checkout still completes and returns the correct order status — the email send failure must not surface as a checkout error
- [ ] notification-service log output shows a message for both the success and failure paths, matching what was (or would have been) emailed
- [ ] Full frontend walkthrough: view a few products, add to cart, fill in the card form, checkout, confirm the payment outcome displays correctly, confirm the email arrives
- [ ] Attempt to hit payment-service (8085) or notification-service (8086) directly, bypassing order-service and api-gateway, with no `X-Internal-Secret` header — confirm 401 from each service's own filter
- [ ] Confirm a full checkout (success path) actually reaches payment-service and notification-service correctly — if order-service's outbound secret header is missing/wrong, checkout would likely fail in a way that could be mistaken for the `.13`-cents mock-rejection rule or a genuine payment failure; cross-check by confirming payment-service's own logs show the request arriving, not just relying on order-service's final response