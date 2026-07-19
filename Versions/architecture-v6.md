# ARCHITECTURE — v6 (OTP Verification)

> Carried over from v5: eureka-server, api-gateway, product-service, order-service, recommendation-service,
> payment-service, notification-service, frontend-service all unchanged except where this file says otherwise.
> See `services-index.md` for the full summary.
>
> New in v6: a single OTP mechanism (generate 6-digit numeric code → email it → store it with an expiry and
> attempt counter → verify → resend-after-30s) applied in two places: user registration and checkout.
> No new service. Two services gain new entities: user-service and order-service.
>
> Verified against the actual `architecture-v2.md`, `architecture-v4.md`, `architecture-v5.md` (patched
> 2026-07-20; the first draft of this file was written from a secondhand summary and got two things wrong —
> see the inline notes in Sections 4.1 and 4.3 for what changed and why). Card details are never sent to
> order-service (v4 Section 8.2/4.4) — the OTP gate sits in front of *all* of v4's checkout logic, not after
> a card-validation step, because order-service never performs one.

---

## 1. Services in this version

| Service | Port | Change from v5 |
|---|---|---|
| eureka-server | 8761 | unchanged |
| api-gateway | 8000 | whitelist gains two new public paths (Section 5) |
| product-service | 8081 | unchanged |
| user-service | 8082 | **registration flow changed** — new entity, new endpoints (Section 3) |
| order-service | 8083 | **checkout flow changed** — new entity, new endpoints (Section 4) |
| recommendation-service | 8084 | unchanged |
| payment-service | 8085 | unchanged |
| notification-service | 8086 | unchanged — no code changes, just a new caller (Section 3) and richer message content (Section 4.4) |
| frontend-service | 8080 | two new pages, two controllers touched (Section 6) |

Global defaults for both OTP flows (registration and checkout) — same numbers, not configurable per-flow in v6:
- OTP: 6-digit numeric, generated via a simple random int in `[100000, 999999]`, stored as a `String` (preserves leading behavior, no leading-zero ambiguity since range starts at 100000).
- Expiry: 5 minutes from generation. An expired OTP cannot be verified — the user must resend.
- Max wrong attempts: 5. On the 5th wrong verify, the row is locked (no more verify attempts accepted) until a resend generates a fresh OTP and resets the counter.
- Resend cooldown: 30 seconds from the last OTP's `generatedAt`. Enforced server-side on every resend call — a disabled frontend button is a UX nicety, not the actual control.

---

## 2. Why two entities, not one shared generic OTP table

Registration-OTP and checkout-OTP hold different contextual payloads (pending *user* details vs. a pending *order's* card/OTP state) and belong to different services with different databases entirely — there is no shared table to begin with. Each gets its own small entity, following the same "keep it a plain, service-local concept" instinct v1 used for `Order.status` (a plain string, not a shared enum type). Same mechanism, two independent implementations.

---

## 3. user-service — registration OTP

### 3.1 New entity: PendingRegistration

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-generated (IDENTITY) |
| name | String | `@NotBlank`, max 60 — copied from the original register request |
| email | String | `@NotBlank`, `@Email` — **not** `@Column(unique = true)` here; uniqueness against real accounts is enforced against `User`, not this table (Section 3.3) |
| passwordHash | String | `@NotBlank` — hashed once, at initial submission, same `BCryptPasswordEncoder` as v1; never re-hashed on verify |
| otp | String | `@NotBlank`, length 6 |
| otpGeneratedAt | LocalDateTime | set on creation and on every resend |
| attempts | Integer | `@NotNull`, default 0 — count of wrong verify attempts since the current OTP was generated |
| createdAt | LocalDateTime | `@PrePersist`, set once, never updated |

7 fields. If a second registration attempt comes in for the same email while a `PendingRegistration` already exists, the existing row is deleted and replaced with a fresh one (fresh OTP, fresh `createdAt`) rather than layering a uniqueness check on top of it — simplest correct behavior for "I didn't get the email, let me try again from the start." No `@PrePersist`-set `role`/other `User`-shaped fields here; those are only assigned when the real `User` row is finally created in 3.2.

### 3.2 Endpoint changes

| Method | Path | Purpose | Notes |
|---|---|---|---|
| POST | `/api/users/register` | **changed** — start registration | body unchanged (name, email, password). No longer creates a `User` row. Checks `existsByEmail` against `User` (existing v1 check, unchanged — still 409 `EmailAlreadyExistsException` if a real account already has this email); if clear, deletes any existing `PendingRegistration` for this email, creates a fresh one with a new OTP, calls notification-service (Section 3.4), returns 200 with `{"email": "...", "message": "OTP sent to your email"}` — **no token, no user id**, since no account exists yet |
| POST | `/api/users/register/verify-otp` | **new** — complete registration | body: `{"email": "...", "otp": "..."}`. Looks up `PendingRegistration` by email (404 `PendingRegistrationNotFoundException` if none). If `attempts >= 5`: 423 `TooManyOtpAttemptsException` ("request a new code"). If `now > otpGeneratedAt + 5min`: 410 `OtpExpiredException`. If `otp` doesn't match: increment `attempts`, save, 400 `InvalidOtpException` with remaining-attempts count in the message. If it matches: create the real `User` row (name, email, passwordHash carried over verbatim, `role` defaulted exactly as v1's `@PrePersist` already does), delete the `PendingRegistration` row, return 201 with the same shape v1's old register success returned |
| POST | `/api/users/register/resend-otp` | **new** — request a fresh code | body: `{"email": "..."}`. Looks up `PendingRegistration` by email (404 if none — nothing pending to resend for). If `now < otpGeneratedAt + 30s`: 429 `OtpResendTooSoonException`. Otherwise: generate a new OTP, reset `attempts` to 0, update `otpGeneratedAt`, re-email it, return 200 with the same shape as the initial register response |

### 3.3 Email uniqueness — unchanged, still two-layer

The `existsByEmail` pre-check plus `DataIntegrityViolationException` backstop in `GlobalExceptionHandler` (established in v1, restated in `promptrule.md`) still applies to `User.email` exactly as before — it runs at the start of `/register`, before a `PendingRegistration` is ever created. Nothing about that pattern changes in v6.

### 3.4 New outbound call: user-service → notification-service

user-service did not call notification-service before v6 (only order-service did, since v4). This is a new Feign client, `NotificationServiceClient`, in `com.ecommerce.userservice.client`, resolved through Eureka the same way order-service's existing clients are, carrying `X-Internal-Secret` on every call (same `FeignInternalSecretInterceptor`-shaped pattern order-service already established in v2 — reuse that pattern here, don't invent a new one per `promptrule.md` Section 2). Reuses notification-service's existing `POST /api/notifications` endpoint unchanged. `NotificationRequest` is `{orderId, email, message}` (confirmed from v4) — `orderId` doesn't apply to a registration email, so user-service's local copy of the request DTO sends `orderId: null`. Nothing in v4's file marks `orderId` `@NotNull` on notification-service's side, so this should pass through cleanly, but flag it explicitly in the user-service prompt so a fresh chat doesn't invent a fake orderId or add unwanted validation on notification-service's side to "fix" a null it was never designed to reject. The OTP code itself is just the `message` text; notification-service's own code needs no changes, same as it doesn't for the checkout path.

---

## 4. order-service — checkout OTP

### 4.1 New entity: CheckoutOtp

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-generated (IDENTITY) |
| order | Order | `@OneToOne`, `@JoinColumn(name = "order_id")`, not nullable — the specific order this OTP gates |
| otp | String | `@NotBlank`, length 6 |
| otpGeneratedAt | LocalDateTime | set on creation and on every resend |
| attempts | Integer | `@NotNull`, default 0 |
| createdAt | LocalDateTime | `@PrePersist` |

5 fields. **No card-data field of any kind** — v4 Section 8.2/4.4 states explicitly that card number/CVV/expiry are validated in the frontend only and are never transmitted to order-service, "not stored in the session, not logged." `CheckoutOtp` must not become the place that boundary quietly gets broken; there is nothing to snapshot here because order-service never receives card data in the first place (see 4.3 — the checkout-start call has no body). Row is deleted once the order moves past the OTP gate (success or the order is otherwise abandoned) — it's a transient holding row, not a permanent audit record.

### 4.2 New Order.status value: OTP_PENDING

Per `promptrule.md`'s rule on preserving existing enum-like values, this is additive only. Existing values — `CART`, `PLACED` (v1), `PENDING_PAYMENT`, `PAID`, `PAYMENT_FAILED` (v4), `PAYMENT_PENDING_RETRY` (v5's circuit-breaker fallback) — are untouched. Confirmed against the actual v4/v5 files: `OTP_PENDING` sits **before v4's checkout logic runs at all**, not after any card step — v4's checkout endpoint takes no body, since card fields are frontend-only and never reach order-service (v4 Section 8.2/4.4). So the OTP gate can't be "after card validation" inside order-service; there's no such step here to gate.

```
CART → [checkout requested, non-empty-cart check passes (existing v1 check, unchanged)]
     → OTP_PENDING
     → [OTP verified]
     → (v4's existing flow, entirely unchanged from here: validate+decrement stock → PENDING_PAYMENT
        → call payment-service → 3.5 resolve email via user-service → 4a PAID + full-bill notification,
        or 4b PAYMENT_FAILED + restock + notification, or v5's circuit-breaker fallback → PAYMENT_PENDING_RETRY)
```

**Required entity-level change, easy to miss (v4 hit this same issue when it added its own new statuses):** `Order`'s `@PrePersist`/`@PreUpdate` lifecycle hooks hardcode which status strings are legal. That hardcoded list must be widened to also accept `OTP_PENDING`, or every v6 checkout will 500 the instant the order is saved with the new status — exactly the failure mode v4 called out explicitly for its own three new statuses. State this in the order-service prompt for this version; don't treat `Order` as merely "gains a new value" without touching the validation itself.

### 4.3 Endpoint changes

| Method | Path | Purpose | Notes |
|---|---|---|---|
| POST | `/api/orders/checkout/{userId}` | **changed** — start checkout | **no body** — unchanged from every prior version; card fields stay entirely frontend-side and are never part of this request, per v4's explicit boundary. Loads the CART order, runs v1's existing empty-cart check (400 if empty, unchanged), and if it passes: order status → `OTP_PENDING`, `CheckoutOtp` row created (OTP only, no card data), notification-service called (Section 4.4), returns 200 `{"orderId": ..., "message": "OTP sent to your email"}`. Does **none** of v4's stock/payment logic yet — all of it, from stock validation onward, is deferred to verify-otp |
| POST | `/api/orders/checkout/{userId}/verify-otp` | **new** — complete checkout | body: `{"otp": "..."}`. Finds the user's `OTP_PENDING` order and its `CheckoutOtp` row (404 if none). Same attempts/expiry rules as 3.2's registration flow (5 max attempts → 423, 5-min expiry → 410, mismatch → 400 with attempts incremented). On match: deletes `CheckoutOtp`, then runs v4's **entire** checkout sequence from its actual first step, unchanged — validate-then-decrement stock, status → `PENDING_PAYMENT`, call payment-service, resolve email via user-service (v4 step 3.5), then 4a (`PAID` + notification) or 4b (`PAYMENT_FAILED` + restock + notification), or v5's circuit-breaker fallback (`PAYMENT_PENDING_RETRY`) if payment-service times out. The success-path notification's content is the richer itemized bill from Section 4.4; failure/retry notifications keep v4's existing short-text messages, unchanged |
| POST | `/api/orders/checkout/{userId}/resend-otp` | **new** — request a fresh code | same 30-second-cooldown pattern as 3.2's registration resend, scoped to the user's `OTP_PENDING` order |

**Ownership — needs an explicit registration change, not just "already covered":** v5's Section 2.4 `HandlerInterceptor` is registered against a fixed list of literal path patterns, one of which is `POST /api/orders/checkout/{userId}`. The two new sub-paths (`.../verify-otp`, `.../resend-otp`) are *not* automatically included just because they share a prefix — they must be added explicitly to the same `WebMvcConfigurer` registration alongside the existing five, or they'll run with no ownership check at all. State this directly in the order-service prompt for this version.

### 4.4 Order-confirmation email — richer content, same call

Confirmed against the real v4 file: `NotificationRequest` is `{orderId, email, message}` — a free-text `message` field already exists, so no DTO change is needed here. Only the **`PAID` path's message content** changes: instead of v4's short success text, order-service builds a full itemized bill — order id, each line item (product name snapshot, quantity, price snapshot), and the total — and sends that as `message`. `PAYMENT_FAILED` and `PAYMENT_PENDING_RETRY` notifications keep v4's/v5's existing short-text messages unchanged; there's no itemized bill to send for an order that didn't succeed. notification-service's own code is untouched — it just sends whatever `message` it's given, same as it already does.

---

## 5. api-gateway change

No new routes — `/api/users/register/**` and `/api/orders/checkout/**` already fall under the existing `user-route`/`order-route` predicates from v2. The only change is to `JwtValidationFilter`'s whitelist:

- `POST /api/users/register/verify-otp` — **public, add to whitelist** (same reasoning as `/api/users/register` itself: no account exists yet, there's no JWT to check).
- `POST /api/users/register/resend-otp` — **public, add to whitelist**, same reasoning.
- `POST /api/orders/checkout/**` (start, verify-otp, resend-otp) — **no whitelist change** — these stay in the authenticated set, exactly like checkout already was in v4/v5. A logged-in user's own JWT is required, and the existing ownership interceptor (Section 4.3) applies unchanged.

---

## 6. frontend-service changes

| Page | Change | Calls |
|---|---|---|
| `register.html` | unchanged form itself; on submit, redirects to new `register-otp.html` instead of a success page | user-service `POST /api/users/register` through the gateway |
| `register-otp.html` | **new** — 6-digit code entry form, plus a "resend code" action. Resend button is disabled client-side for 30s after each send (small, deliberate exception to v1's "no JS" rule — a plain countdown display only; the actual 30-second rule is still enforced server-side, the button being disabled early is UX polish, not the control) | user-service `POST /api/users/register/verify-otp`, `POST /api/users/register/resend-otp` |
| `cart.html` → checkout | unchanged from v4 — the card-entry form (Section 8.2 of v4) still validates locally and its fields still go nowhere beyond that validation, per v4's explicit boundary. Once validation passes, the frontend calls order-service's checkout-start endpoint with no body (same as today), then redirects to new `checkout-otp.html` instead of straight to `order-confirmation.html` | order-service `POST /api/orders/checkout/{userId}` (no body) through the gateway |
| `checkout-otp.html` | **new** — same shape as `register-otp.html`: code entry + 30s-gated resend | order-service `POST /api/orders/checkout/{userId}/verify-otp`, `.../resend-otp` |
| `order-confirmation.html` | **changed** — now renders the itemized bill (line items, quantities, prices, total) matching 4.4's richer email content, instead of v1's plain confirmation | rendered from the response of `verify-otp`, no new call |

New frontend DTOs: `RegisterOtpForm`, `CheckoutOtpForm` (both just `{otp: String}`, possibly `{email}` / implicit userId-from-session respectively). `AuthController` gains the two register-OTP routes; the checkout controller (`OrderController`, per the manifest's Part B naming) gains the two checkout-OTP routes. No new exception classes expected beyond translating the new 400/410/423/429 responses into user-facing error messages on the same OTP-entry template — reuse `WebExceptionHandler`'s existing pattern.

---

## 7. Known limitations (intentional, fixed in later versions or accepted trade-offs)

- No cleanup job for abandoned `PendingRegistration` or `CheckoutOtp` rows (user never completes the OTP step) — they just sit. A repeat attempt with the same email/order naturally overwrites or supersedes the stale row, consistent with the project's existing tolerance for this class of gap (see v4's accepted user/product orphan gaps).
- OTP delivery itself is not verified/retried — if notification-service's send silently fails, the user only discovers it via "resend" after 30 seconds, same fail-open spirit as recommendation-service's v3 behavior, just not formalized with the same language.
- No rate-limiting beyond the per-flow 30-second resend cooldown and 5-attempt lock — no IP-level or account-level throttling across multiple different emails/orders. Out of scope for this version.

---

## 8. Testing checklist before calling v6 "done"

- [ ] Register with a new email → no `User` row created yet, `PendingRegistration` row exists, OTP email received
- [ ] Submit correct OTP → `User` row created, `PendingRegistration` row deleted, can log in immediately after
- [ ] Submit wrong OTP → 400, `attempts` increments; after 5 wrong attempts → 423, further verify attempts rejected until resend
- [ ] Wait past 5 minutes, submit correct OTP → 410 (expired), must resend
- [ ] Click resend before 30s have passed → 429; after 30s → succeeds, new OTP received, `attempts` reset to 0
- [ ] Registering the same email twice before verifying the first → old `PendingRegistration` replaced, only the newest OTP works
- [ ] Attempt registering with an email that already has a real `User` account → 409, unchanged from v1
- [ ] Submit checkout on a non-empty cart → order status `OTP_PENDING`, no stock decrement yet, no payment-service call yet (confirm via product-service that stock is untouched), OTP email received, no card fields present anywhere in the checkout-start request body
- [ ] Submit checkout on an empty cart → 400, exactly as v1, no OTP sent, no `CheckoutOtp` row created
- [ ] Submit correct checkout OTP → v4's full flow now runs (stock validates and decrements, status → `PENDING_PAYMENT`, payment-service called, order reaches `PAID`/`PAYMENT_FAILED`, or `PAYMENT_PENDING_RETRY` if payment-service is down), `CheckoutOtp` row deleted
- [ ] `PAID` path's email/confirmation page shows the full itemized bill; `PAYMENT_FAILED`/`PAYMENT_PENDING_RETRY` paths still show v4's/v5's existing short message, unchanged
- [ ] Wrong/expired/too-many-attempts checkout OTP behave identically to the registration checks above (400/410/423)
- [ ] Checkout OTP resend follows the same 30s cooldown
- [ ] Final confirmation email (and `order-confirmation.html`) shows the full itemized bill, not the old one-line message
- [ ] Attempt to hit `verify-otp`/`resend-otp` for another user's `{userId}` while logged in as someone else → blocked by v5's existing ownership interceptor, unchanged
- [ ] `POST /api/users/register/verify-otp` and `resend-otp` work through the gateway with **no** token present; `POST /api/orders/checkout/**` OTP endpoints correctly return 401 without a token