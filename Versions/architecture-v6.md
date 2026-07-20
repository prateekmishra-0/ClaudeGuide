# ARCHITECTURE — v6 (OTP Verification: Registration & Checkout)

> Carried over from v5: all 8 services exist and are unchanged in shape — see `services-index.md` for the summary. No new services in v6. This version adds a one-time-password gate in front of two existing flows (registration in user-service, checkout in order-service), reusing notification-service's existing email capability rather than adding a new channel. It also upgrades the checkout notification email from v4's one-line sentence to a full bill summary.
> This version makes two deliberate, named **breaking changes** to existing endpoint contracts: `POST /api/users/register` no longer creates a user immediately, and `POST /api/orders/checkout/{userId}` (the single-step version) is retired outright, replaced by a three-step flow. Both are called out explicitly in their own sections below — neither is a silent rename.
> Seller role (product ownership, seller panel) is explicitly **out of scope for this file** — see `architecture-v7.md`.

---

## 1. What's added (by service)

| Service | Addition |
|---|---|
| user-service | Registration becomes a two-step, OTP-gated flow: new `PendingRegistration` entity, three new endpoints, new outbound call to notification-service (Section 3) |
| order-service | Checkout becomes a three-step, OTP-gated flow: new `CheckoutOtp` entity, three new endpoints replacing the old single-step one, richer bill-summary email content (Section 4, Section 6) |
| notification-service | New endpoint dedicated to OTP emails, separate from the existing order-outcome endpoint (Section 5) |
| api-gateway | Two new whitelist entries (registration's verify/resend), no new routes needed (Section 7) |
| frontend-service | New OTP-entry pages for both registration and checkout, Feign client method changes to match the new backend contracts (Section 8) |

---

## 2. Shared OTP pattern

Both services implement the same rules independently — **not shared code**, since user-service and order-service are separate projects with separate databases (same convention already established for `ForbiddenException` in v5, implemented once per service rather than shared).

| Rule | Value |
|---|---|
| OTP format | 6 digits, numeric, zero-padded (e.g. `007421`), generated via `java.security.SecureRandom` — **not** `Math.random()`. This doesn't contradict payment-service's v4 "no `Math.random()`" rule, which was about deterministic *testability* of a business outcome; OTP generation is a security-sensitive value that must be genuinely unpredictable, a different concern entirely. |
| Expiry | 5 minutes from `otpGeneratedAt` |
| Wrong-attempt cap | 5 incorrect verify attempts. On the 5th wrong attempt, the pending record is deleted and the caller must start over (re-register / re-initiate checkout), not just resend. |
| Resend cooldown | 30 seconds since `otpGeneratedAt`, enforced **server-side** on every resend call — this is the actual security boundary, not a UI convenience. A resend does **not** reset the wrong-attempt counter — only a full restart does. |
| Storage | Both new entities store the OTP as `String`, length 6, never as an `Integer` (preserves leading zeros) |

---

## 3. user-service — registration OTP

### 3.1 New entity: `PendingRegistration` (package `com.ecommerce.userservice.entity`)

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-generated (IDENTITY) |
| name | String | `@NotBlank`, max 60 |
| email | String | `@NotBlank`, `@Email`, `@Column(unique = true)` — one pending registration per email; a repeat call to `/register` for the same email overwrites (delete + recreate) the existing pending row rather than creating a second one |
| passwordHash | String | `@NotBlank` — hashed via `BCryptPasswordEncoder` at initiate time, exactly as v1 did at final creation time; never store plaintext even transiently |
| otp | String | `@NotBlank`, `@Column(length = 6)` |
| otpGeneratedAt | LocalDateTime | set at initiate and again at every resend, not client-settable |
| attempts | Integer | defaults 0, incremented on each wrong verify attempt |

New repository: `PendingRegistrationRepository extends JpaRepository<PendingRegistration, Long>`, one custom method `Optional<PendingRegistration> findByEmail(String email)`.

### 3.2 Endpoints

| Method | Path | Purpose | Notes |
|---|---|---|---|
| POST | `/api/users/register` | **initiate** registration | body: `RegisterRequest` (name, email, password) — same shape as v1, unchanged. **Behavior change from v1–v5:** no longer creates the `User` row. Validates the email isn't already a real registered user (existing `existsByEmail` check, still 409 `EmailAlreadyExistsException` if so — unchanged), hashes the password, generates an OTP, upserts `PendingRegistration` (overwriting any existing pending row for that email), calls notification-service (Section 3.5), returns **200** with `OtpSentResponse{message, email}` — not the old 201-with-`UserResponse`. |
| POST | `/api/users/register/verify` | finalize registration | body: `OtpVerifyRequest{email, otp}`. Look up `PendingRegistration` by email → 404 `PendingRegistrationNotFoundException` if none. If `now - otpGeneratedAt > 5 min` → 400 `OtpExpiredException`. If `otp` doesn't match → increment `attempts`; if now ≥ 5 → delete the pending row, 429 `TooManyAttemptsException` ("too many incorrect attempts, please register again"); else 400 `InvalidOtpException`. If it matches → create the real `User` row (role defaults `"CUSTOMER"` via the existing `@PrePersist`, unchanged since v1), delete the `PendingRegistration` row, return **201** `UserResponse` — same shape v1's original `/register` used to return. |
| POST | `/api/users/register/resend-otp` | resend a registration OTP | body: `OtpResendRequest{email}`. Look up `PendingRegistration` by email → 404 if none. If < 30s since `otpGeneratedAt` → 429 `ResendTooSoonException`. Else generate a new OTP, update `otpGeneratedAt` (attempts **not** reset), call notification-service again, return 200 `OtpSentResponse`. |

### 3.3 New exceptions (package `com.ecommerce.userservice.exception`)

`PendingRegistrationNotFoundException` → 404, `InvalidOtpException` → 400, `OtpExpiredException` → 400, `TooManyAttemptsException` → 429, `ResendTooSoonException` → 429. Each wired individually into the existing `GlobalExceptionHandler`, same pattern as every prior custom exception in this project — no shared parent exception class.

### 3.4 New DTOs (package `com.ecommerce.userservice.dto`)

- `OtpVerifyRequest`: email (`@Email`, `@NotBlank`), otp (`@NotBlank`, `@Pattern(regexp = "^\\d{6}$")`)
- `OtpResendRequest`: email (`@Email`, `@NotBlank`)
- `OtpSentResponse`: message (String), email (String)

### 3.5 New outbound call: user-service → notification-service

This is **new for user-service** — it has never called another service before v6 (contrast: order-service has done this since v1). Required additions:
- `@EnableFeignClients` added to `UserServiceApplication` (not previously needed)
- New `NotificationServiceClient` interface (package `com.ecommerce.userservice.client`), `@FeignClient(name = "notification-service")`, one method targeting the new `POST /api/notifications/otp` (Section 5.2)
- New `FeignInternalSecretInterceptor` (package `com.ecommerce.userservice.config`) — same mechanism order-service's copy has had since v4, attaching `X-Internal-Secret` to every outbound call this client makes, resolved directly via Eureka, **not** through the gateway — same "internal calls bypass the gateway" convention every inter-service call in this project has followed since v1

### 3.6 What this does not change

`POST /api/users/login` and `GET /api/users/me` are completely unchanged. `EmailAlreadyExistsException`'s 409 behavior at initiate time is unchanged from v1. The `User` entity itself gains no new fields.

---

## 4. order-service — checkout OTP

### 4.1 New entity: `CheckoutOtp` (package `com.ecommerce.orderservice.entity`)

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-generated (IDENTITY) |
| userId | Long | `@NotNull`, `@Column(unique = true)` — one pending checkout-OTP per user at a time |
| otp | String | `@NotBlank`, `@Column(length = 6)` |
| otpGeneratedAt | LocalDateTime | set at initiate and each resend |
| attempts | Integer | defaults 0 |

New repository: `CheckoutOtpRepository extends JpaRepository<CheckoutOtp, Long>`, one custom method `Optional<CheckoutOtp> findByUserId(Long userId)`.

### 4.2 Endpoints — replacing the old single-step checkout

| Method | Path | Purpose | Notes |
|---|---|---|---|
| POST | `/api/orders/checkout/{userId}/initiate` | begin checkout | **Replaces** v1–v5's `POST /api/orders/checkout/{userId}`, retired in v6 (Section 4.6). Validates the cart is non-empty (400 `EmptyCartException`, unchanged rule from v1) but does **not** touch stock or call payment-service yet. Resolves the user's email via the existing `UserServiceClient` (v4) `/api/users/{id}/email` call. Generates an OTP, upserts `CheckoutOtp` (overwriting any existing pending row for this `userId`), calls notification-service (Section 5.2), returns 200 `OtpSentResponse`. |
| POST | `/api/orders/checkout/{userId}/verify-otp` | finalize checkout | body: `CheckoutOtpVerifyRequest{otp}`. Look up `CheckoutOtp` by `userId` → 404 `CheckoutOtpNotFoundException` if none (client should call `/initiate` first). Expiry/attempts/wrong-otp handling identical in shape to user-service's verify endpoint (Section 3.2), using order-service's own separately-implemented exception classes (Section 4.3). On a correct, non-expired OTP: delete the `CheckoutOtp` row, then run the **existing, completely unchanged** v2/v4/v5 checkout logic — stock decrement, the v5 circuit-breaker-wrapped call to payment-service, `PAID`/`PAYMENT_FAILED`/`PAYMENT_PENDING_RETRY` branching, and the final notification call (now using the bill-summary content from Section 6). This endpoint is a new front door onto logic that already existed — not a rewrite of it. Returns the same `OrderResponse` shape checkout has always returned. |
| POST | `/api/orders/checkout/{userId}/resend-otp` | resend a checkout OTP | body: none (`userId` from path). Same 30-second-cooldown rule as user-service's resend endpoint; `attempts` not reset. Returns 200 `OtpSentResponse`. |

### 4.3 New exceptions (package `com.ecommerce.orderservice.exception`)

`CheckoutOtpNotFoundException` → 404, `InvalidOtpException` → 400, `OtpExpiredException` → 400, `TooManyAttemptsException` → 429, `ResendTooSoonException` → 429 — a separate set from user-service's (Section 3.3), same shape convention, no shared class.

### 4.4 New DTOs (package `com.ecommerce.orderservice.dto`)

- `CheckoutOtpVerifyRequest`: otp (`@NotBlank`, `@Pattern(regexp = "^\\d{6}$")`)
- `OtpSentResponse`: message (String) — order-service's own copy, same shape as user-service's, separately declared

### 4.5 Interaction with v5's ownership interceptor (Section 2.4)

The three new `{userId}`-scoped paths above replace the old single pattern in the interceptor's registered path list:
- `POST /api/orders/checkout/{userId}/initiate`
- `POST /api/orders/checkout/{userId}/verify-otp`
- `POST /api/orders/checkout/{userId}/resend-otp`

No change to the interceptor's logic itself. The v5 admin bypass (`X-User-Role: ADMIN` skips the mismatch check) carries forward unchanged onto all three — this is inherited behavior from how v5 originally scoped the bypass across order-service generally, not a new decision made in this version.

### 4.6 Named breaking change

`POST /api/orders/checkout/{userId}` (the v1–v5 single-step endpoint) is **removed**, not deprecated-and-kept. Any test, client, or checklist item referencing that exact contract (including several items in `architecture-v5.md` Section 10) needs updating to the new three-step flow. This is stated explicitly per `promptrule.md`'s consistency rules — it is a deliberate scope decision for this version, not an oversight.

### 4.7 What this does not add

No re-validation of cart contents beyond what the existing stock-check inside checkout logic already does naturally by operating on the live cart at verify-time — no new snapshot or lock is taken between initiate and verify. If the cart changes in that window, the existing v2 stock-check behaves exactly as it always has.

---

## 5. notification-service — OTP email endpoint

### 5.1 New DTO (package `com.ecommerce.notificationservice.dto`)

`OtpEmailRequest`: email (`@Email`, `@NotBlank`), otp (`@NotBlank`, `@Pattern(regexp = "^\\d{6}$")`), purpose (`@NotBlank` — either `"REGISTRATION"` or `"CHECKOUT_VERIFICATION"`, used only to vary the email subject line, not stored, not branched into different mail-sending code paths beyond that one string)

### 5.2 New endpoint

| Method | Path | Purpose | Notes |
|---|---|---|---|
| POST | `/api/notifications/otp` | send an OTP email | body: `OtpEmailRequest`, `@Valid`. Builds a `SimpleMailMessage`: `from` = configured `spring.mail.username` (unchanged v4 pattern), `to` = `email`, subject = `"Verify your registration"` or `"Confirm your payment"` depending on `purpose`, text = `"Your OTP is " + otp + ". It is valid for 5 minutes. If you did not request this, you can safely ignore this email."` Same try/catch-log-never-throw pattern as v4's existing `sendNotification` — always returns **201** regardless of actual send success, same INFO-level logging discipline. No new exception, no `GlobalExceptionHandler` entry — same "nothing else for one to catch" reasoning v4 already stated still holds. |

### 5.3 Existing endpoint — unchanged, but fed differently

`POST /api/notifications` (v4) is **not modified** in this version. Its `message` field is still a plain `String` — what changes is what order-service puts into that string at the existing checkout-completion call sites (Section 6). No DTO change, no notification-service code change for this part.

---

## 6. Bill-summary email content (order-service change only)

Where `OrderService` (v4) already builds a `NotificationRequest` at the end of a successful or failed checkout, the `message` field's construction changes from a one-line sentence to a multi-line plain-text summary. `SimpleMailMessage` supports multi-line text natively — no notification-service change needed, this is string-building inside order-service only.

**On `PAID`:**
```
Order #<id> — Payment successful

Items:
- <productNameSnapshot> x<quantity> @ $<priceSnapshot> = $<lineTotal>
- ...

Total: $<total>

Thank you for your order.
```

**On `PAYMENT_FAILED`:**
```
Order #<id> — Payment failed

Your payment could not be processed. The items below have been returned to stock and you have not been charged.

Items:
- <productNameSnapshot> x<quantity> @ $<priceSnapshot>
- ...

Attempted total: $<total>

Please try checking out again.
```

`PAYMENT_PENDING_RETRY` (v5): notification behavior for this status is unchanged from v5 — this section only touches the message content of the two call sites v4 already had (PAID, PAYMENT_FAILED). Whether a pending-retry notification exists at all was out of scope in v5 and stays out of scope here.

---

## 7. api-gateway

No new routes — the existing `user-route` (`Path=/api/users/**`) and `order-route` (`Path=/api/orders/**`) already match every new path in this version.

**Whitelist changes (`JwtValidationFilter`):**
- Add `POST /api/users/register/verify` and `POST /api/users/register/resend-otp` to the public whitelist, alongside the existing `POST /api/users/register` and `POST /api/users/login` — a user isn't authenticated yet during registration, so these must stay open, same reasoning as every whitelist entry since v2.
- The three new order-service checkout endpoints (`initiate`, `verify-otp`, `resend-otp`) are **not** whitelisted — same authenticated-only status the old single checkout endpoint always had.

---

## 8. frontend-service

### 8.1 New DTOs (package `com.ecommerce.frontendservice.dto`)

- `RegisterOtpForm`: email (String), otp (String, `@NotBlank`, `@Pattern(regexp = "^\\d{6}$")`)
- `CheckoutOtpForm`: otp (String, `@NotBlank`, `@Pattern(regexp = "^\\d{6}$")`)

### 8.2 Feign client changes

**`UserServiceClient`** — the existing method targeting `POST /api/users/register` changes its return type from `UserResponse` to `OtpSentResponse` (named breaking change, matches Section 3.2). Add two new methods: `POST /api/users/register/verify` (returns `UserResponse`), `POST /api/users/register/resend-otp` (returns `OtpSentResponse`).

**`OrderServiceClient`** — the existing `checkout(userId)` method is retired and replaced with `initiateCheckout(userId)` targeting `POST /api/orders/checkout/{userId}/initiate` (returns `OtpSentResponse`). Add: `verifyCheckoutOtp(userId, CheckoutOtpVerifyRequest)` targeting `.../verify-otp` (returns `OrderResponse`), `resendCheckoutOtp(userId)` targeting `.../resend-otp` (returns `OtpSentResponse`).

### 8.3 `AuthController` changes (registration)

- `POST /register`: now calls `userServiceClient.initiateRegistration(request)`. On success, add `email` to the Model, return view `register-verify-otp` (new template) instead of the old redirect-to-login. On 409 (`EmailAlreadyExistsException`, unchanged exception) → redisplay `register.html` with the error, exactly as before.
- New `POST /register/verify-otp`: reads `RegisterOtpForm` (email carried as a hidden field, otp entered by the user) → calls `userServiceClient.verifyRegistrationOtp(...)`. On success → redirect to `/login` with a flash message, "Registration complete — please log in." On `InvalidOtpException`/`OtpExpiredException` (400) → redisplay `register-verify-otp.html` with an error, keeping the email hidden field. On `TooManyAttemptsException` (429) → redisplay with a distinct message, "Too many incorrect attempts — please register again," plus a plain link back to `/register` (no auto-redirect — this project has never used JavaScript, since v1, and that constraint is unchanged here).
- New `POST /register/resend-otp`: reads the hidden `email` field → calls `userServiceClient.resendRegistrationOtp(email)` → redisplays `register-verify-otp.html` with either a "a new code has been sent" confirmation or, on `ResendTooSoonException` (429), a "please wait a little longer before requesting another code" message. Plain server round trip — no live countdown timer, since that would require JavaScript. The 30-second rule (Section 2) is enforced by the server on every click; the page simply tells the user if they clicked too soon.

### 8.4 `OrderController` changes (checkout)

- Existing `POST /checkout` (card form submit, v4): after `CheckoutForm` validation passes (unchanged Bean Validation logic), replace the call to `orderServiceClient.checkout(userId)` with `orderServiceClient.initiateCheckout(userId)`. On success → render new view `checkout-otp`, re-adding `cart`/`cartTotal` to the Model (same values `GET /checkout` already computes) plus a blank `CheckoutOtpForm` — this does **not** go straight to `order-confirmation` anymore.
- New `POST /checkout/verify-otp`: reads `CheckoutOtpForm` → calls `orderServiceClient.verifyCheckoutOtp(userId, form)`. On success → same existing v4 success path, return view `order-confirmation` with `order` in the Model. On the existing `FeignException.Conflict`/`FeignException.BadRequest` mappings (`InsufficientStockException`/`EmptyCartException` — still possible, since stock is only actually touched now, at verify-time) → same existing flash-error-and-redirect-to-`/cart` behavior, unchanged from v4. On `InvalidOtpException`/`OtpExpiredException` → redisplay `checkout-otp.html` with an error, re-fetching `cart`/`cartTotal` (same re-fetch-on-redisplay pattern the existing card-validation-failure branch already uses). On `TooManyAttemptsException` → redisplay with "too many attempts — please start checkout again" plus a plain link back to `/checkout` (which cleanly re-initiates, since `GET /checkout` already guards for an empty cart).
- New `POST /checkout/resend-otp`: calls `orderServiceClient.resendCheckoutOtp(userId)` → redisplays `checkout-otp.html` with a confirmation or wait-message, same pattern as registration's resend.

### 8.5 New templates

`register-verify-otp.html`, `checkout-otp.html`. Same header-fragment inclusion and no-JavaScript constraint as every template since v1. The resend action is a plain link/button posting to the resend endpoint — no inline countdown; a too-early click simply redisplays the page with the server's message, the same way every other validation error in this project already surfaces.

`order-confirmation.html` needs no structural change — it already displays order id, items, and status per v4/v5. The bill-summary email (Section 6) is a separate artifact delivered to the user's inbox, not something the page itself renders differently.

---

## 9. Known limitations (intentional, not solved here)

- `PendingRegistration` and `CheckoutOtp` rows are never cleaned up by a background job. An abandoned attempt just sits until overwritten by a fresh initiate call for the same email/userId. Accepted, consistent with this project's existing tolerance for this class of gap (same shape as v5's orphaned-orders-after-user-deletion gap).
- OTP email delivery failures are silent, inherited from notification-service's fail-open policy since v4. If an email genuinely never arrives, the user's only recourse is Resend — there's no delivery confirmation or alternate channel.
- No CAPTCHA or IP-based rate limiting. The 30-second cooldown and 5-attempt cap are per-pending-record (per email, per userId), not per-client — a scripted caller could still spread requests across many different identities. Accepted at this project's scale.
- The old `POST /api/orders/checkout/{userId}` and the old immediate-user-creation behavior of `POST /api/users/register` are both retired (Sections 4.6, 3.2) — this is a deliberate, named breaking change, not an oversight, and any prior checklist item referencing the old contracts needs updating.

---

## 10. Testing checklist before calling v6 "done"

- [ ] `POST /api/users/register` with a new email returns 200 `OtpSentResponse`, **not** 201 — confirm no `User` row exists yet in the database
- [ ] The registration OTP email actually arrives, with the correct 6-digit code
- [ ] `POST /api/users/register/verify` with the correct OTP returns 201 `UserResponse`, and the `User` row now exists; the `PendingRegistration` row is gone
- [ ] `POST /api/users/register/verify` with a wrong OTP returns 400, and doing so 5 times in a row returns 429 on the 5th and deletes the pending row — confirm a 6th attempt (even with the right code) now returns 404
- [ ] `POST /api/users/register/verify` after waiting 6 minutes past the original OTP returns 400 `OtpExpiredException`
- [ ] `POST /api/users/register/resend-otp` called twice within 30 seconds returns 429 on the second call; called again after 30+ seconds succeeds and the new code (not the old one) is required to verify
- [ ] `POST /api/users/register` for an email that's already a real registered user still returns 409, unchanged from v1
- [ ] Add an item to cart, proceed to checkout, submit a valid card → confirm you land on the new OTP page, **not** `order-confirmation`, and confirm no stock has been decremented yet
- [ ] The checkout OTP email actually arrives
- [ ] Submitting the correct checkout OTP proceeds exactly as v4/v5's checkout always did — stock decrements, payment-service is called, `order-confirmation` renders — and the email received is now the full bill summary (Section 6), not the old one-line sentence
- [ ] Submitting a wrong checkout OTP, an expired one, and 5 wrong attempts in a row all behave the same way as the registration flow's equivalents above
- [ ] Checkout's resend-OTP 30-second cooldown behaves the same way as registration's
- [ ] Confirm `POST /api/orders/checkout/{userId}/initiate` (and the other two new checkout endpoints) still reject a mismatched `{userId}` for a CUSTOMER-role token with 403, and still allow it for an ADMIN-role token, per Section 4.5
- [ ] Confirm `POST /api/users/register/verify` and `/resend-otp` work through the gateway with **no** Authorization header (they're whitelisted); confirm the checkout OTP endpoints still return 401 through the gateway with no token
- [ ] Attempt `POST /api/orders/checkout/{userId}` (the old v1–v5 path) — confirm it now 404s at the gateway/service level, since the route pattern still matches but no controller method handles it anymore
- [ ] Full end-to-end walkthrough: register → verify OTP → log in → browse → add to cart → checkout → verify OTP → land on order-confirmation → check inbox for the full bill-summary email — zero unhandled errors anywhere in the chain
