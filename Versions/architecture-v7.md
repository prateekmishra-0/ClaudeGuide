# ARCHITECTURE — v7 (OTP Verification)

> Carried over from v6: all 8 services exist — eureka-server, api-gateway, product-service, user-service, order-service, recommendation-service, payment-service, notification-service, frontend-service — see `services-index.md` for the summary. This file covers only what's new for v7: OTP-gated registration, OTP-gated checkout, and the bill-summary email upgrade. Nothing about v6's seller work (role selector, `sellerId` ownership, cascade delete) changes.
>
> Not in scope for this version: no OTP on login — v6's `loginAs` selector and role-mismatch check stay exactly as they are. No OTP anywhere except registration and checkout.

---

## 1. Services in this version

| Service | Port | Change from v6 |
|---|---|---|
| eureka-server | 8761 | unchanged |
| api-gateway | 8000 | no new routes — existing `user-route` and `order-route` predicates already match the new sub-paths; the JWT filter's whitelist entries and the ownership interceptor's path list are updated (Section 6) |
| product-service | 8081 | unchanged |
| user-service | 8082 | registration becomes a 3-step OTP flow; gains its **second** outbound Feign client (to notification-service) (Section 3) |
| order-service | 8083 | checkout becomes a 3-step OTP flow, wrapping the existing v4 checkout logic instead of replacing it; bill-summary email upgrade (Section 4) |
| recommendation-service | 8084 | unchanged |
| payment-service | 8085 | unchanged |
| notification-service | 8086 | no entity/contract change — same `{orderId, email, message}` shape, just two new callers of the existing endpoint and richer `message` content for the checkout case (Section 5) |
| frontend-service | 8080 | registration and checkout each gain an OTP-entry step; no JavaScript anywhere, per this project's standing rule (Section 7) |

No new services. No new Postgres tables. Everything OTP-related is held in-memory, per your call — this project won't exceed ~10 concurrent users, so persistence isn't needed for something this short-lived and low-value.

---

## 2. Shared OTP mechanism

This spec is identical for both the registration flow (Section 3) and the checkout flow (Section 4) — one set of rules, applied twice.

| Aspect | Rule |
|---|---|
| Format | 6 random numeric digits, generated as a zero-padded string (`String.format("%06d", ...)`) — never a raw `int`, so `048392` stays 6 characters, not silently truncated to 5 |
| Validity window | 5 minutes from the moment it was last sent |
| Multiple sends | **Last-sent-OTP-only.** A resend overwrites the previous code and its expiry in place — there is never more than one live code for a given pending registration/checkout at once. This is the simpler of the two options considered and also the more correct UX: requesting a resend means the old code no longer counts. |
| Attempt cap | 5 wrong guesses. Checked at verify time. |
| Attempt counter reset | **Only a fresh `initiate` call resets it to 0.** Resend does *not* touch the counter — so someone can't dodge the 5-attempt cap by spamming resend. |
| Resend cooldown | 30 seconds since the OTP was last sent (whether that send was the original `initiate` or a prior `resend`). Enforced server-side only — there is no JavaScript anywhere in this project, so there's no ticking countdown. An early resend click just re-renders the same page with a "please wait" message. |
| Expired code | Checked at verify time as a **distinct** outcome from "wrong code" — if `now > expiresAt`, reject with a specific "OTP expired, please resend" message. This does **not** count against the 5-attempt cap (an expired code isn't a wrong guess). |
| Storage | In-memory only, per service, no persistence — a `ConcurrentHashMap`, same in-memory spirit as v1's original session-token map. No hashing of the OTP itself (unlike passwords) — it's short-lived, low-value, and this is a demo project. |

### 2.1 Shared state shape

Both pending-flow maps (Section 3.2, Section 4.2) hold the same three OTP-related fields per entry, regardless of what else the entry carries:

```java
private String otp;               // "048392" — always 6 chars
private LocalDateTime lastSentAt; // updated on initiate AND on every resend
private LocalDateTime expiresAt;  // lastSentAt + 5 minutes, updated together with lastSentAt
private int attemptCount;         // 0..5, reset only by a fresh initiate
```

`expiresAt` and `lastSentAt` are always updated together (a resend refreshes both the 5-minute validity window and the 30-second cooldown clock at once) — there's no scenario where one moves without the other.

---

## 3. user-service — registration becomes a 3-step OTP flow

### 3.1 What's retired

`POST /api/users/register` (single-step, v1–v6) is **fully retired**, not kept alongside the new flow. This is a real breaking change, same treatment as checkout's retirement in Section 4.

### 3.2 New in-memory store: `PendingRegistration`

```java
ConcurrentHashMap<String email, PendingRegistrationEntry>
```

Keyed by email (the natural key before an account exists). Each entry:

| Field | Type | Notes |
|---|---|---|
| name | String | as submitted |
| email | String | the map key, also stored on the entry for convenience |
| passwordHash | String | hashed at `initiate` time (BCrypt, same as always) — never stored or logged in plaintext, even transiently |
| role | String | `"CUSTOMER"` or `"SELLER"`, validated exactly as v6's Section 3.2 already validates it — `"ADMIN"` or anything else is rejected with 400 before a pending entry is even created |
| otp, lastSentAt, expiresAt, attemptCount | — | per Section 2.1 |

The actual `User` row is **not created** until `verify` succeeds. Everything v6 already does at registration time (email-uniqueness check, password hashing, the `role` validation, the `@PrePersist` default-to-`CUSTOMER` behavior) still happens — it just happens at `verify`, not `initiate`.

### 3.3 New endpoints

| Method | Path | Body | Behavior |
|---|---|---|---|
| POST | `/api/users/register/initiate` | `{name, email, password, role?}` | Existing v1 email-uniqueness check against the real `User` table (409 if taken) — a pending, unverified registration for the same email doesn't block a *different* email, but re-initiating with the *same* email overwrites the existing pending entry outright (a full restart: new OTP, `attemptCount` reset to 0, fresh `lastSentAt`/`expiresAt`). Hashes the password, validates `role` (3.2), stores the `PendingRegistration` entry, sends the OTP via notification-service (Section 3.4), returns 200 — "OTP sent to your email." |
| POST | `/api/users/register/verify` | `{email, otp}` | Looks up the pending entry by email — missing entry → 404 ("no pending registration for this email, please initiate again"). Otherwise: expired (`now > expiresAt`) → distinct error, does **not** increment `attemptCount`. Wrong code → `attemptCount++`; if this reaches 5 → reject and require a fresh `initiate` (the entry can be deleted at this point, or just left to be overwritten by the next `initiate` — deleting it is the cleaner choice). Correct code → create the actual `User` row (existing registration logic, unchanged internals), remove the pending entry, return 201. The person still has to log in separately afterward via the existing `POST /api/users/login` — verify does **not** issue a JWT itself, no change to login's shape. |
| POST | `/api/users/register/resend` | `{email}` | Missing pending entry → 404. Cooldown not yet elapsed (`now - lastSentAt < 30s`) → 429, "please wait." Otherwise: generate a new OTP, overwrite `otp`/`lastSentAt`/`expiresAt` on the existing entry, **leave `attemptCount` untouched**, send it (Section 3.4), return 200. |

### 3.4 New outbound call: user-service → notification-service

This is user-service's **second** Feign client — not its first. v6 already gave user-service its first-ever outbound call (to product-service, for the seller cascade-delete) along with a `FeignInternalSecretInterceptor` that "applies automatically to `ProductServiceClient` (and any future Feign client user-service gains) with no per-call wiring" (v6 Section 3.4). That sentence is exactly what pays off here: no new interceptor is needed, just a new client.

- New Feign client, `NotificationServiceClient` (package `com.ecommerce.userservice.client`), `@FeignClient(name = "notification-service")`.
    - One method: `@PostMapping("/api/notifications") void sendNotification(@RequestBody NotificationRequest request)` — same request shape notification-service has accepted since v4 (`{orderId, email, message}`).
    - For registration OTPs, `orderId` has no natural value — send `null` (the more honest choice, since there genuinely is no order; notification-service doesn't validate or use `orderId` beyond logging it, so this is harmless).
    - `message` is a fixed template: `"Your registration OTP is {otp}. It expires in 5 minutes."`
- No new interceptor — the existing `FeignInternalSecretInterceptor` from v6 already attaches `X-Internal-Secret` to every outbound Feign call user-service makes, including this new one, automatically.
- No `X-User-Id` on this call, same reasoning as v6's product-service call — it's a direct Feign-to-Eureka call, never routed through the gateway.

### 3.5 What doesn't change

- `POST /api/users/login` — untouched. Still v6's shape (`email`, `password`, `loginAs`), still 401 on bad password, 403 on role mismatch, no OTP step.
- `GET /api/users/me`, `GET /api/users/{id}/email` — untouched.
- v6's seller ownership rules, `sellerId` handling, cascade delete — untouched.

---

## 4. order-service — checkout becomes a 3-step OTP flow

### 4.1 What's retired

`POST /api/orders/checkout/{userId}` (single-step, v1–v6) is **fully retired**, not kept alongside the new flow.

### 4.2 New in-memory store: `PendingCheckout`

```java
ConcurrentHashMap<Long userId, PendingCheckoutEntry>
```

Keyed by `userId` (mirrors how the old endpoint was already keyed). Each entry only needs the shared OTP fields from Section 2.1 — no cart contents are duplicated here, since `verify` re-loads the live `CART` order fresh, exactly as the old atomic endpoint always did.

### 4.3 New endpoints

| Method | Path | Behavior |
|---|---|---|
| POST | `/api/orders/checkout/initiate/{userId}` | Loads the `CART` order — empty cart → 400, same as v1's existing guard, unchanged. Otherwise: resolves the user's email via the existing `GET /api/users/{userId}/email` Feign call (unchanged from v4), generates an OTP, stores/overwrites the `PendingCheckout` entry for this `userId` (full restart semantics, same as registration's `initiate` — new OTP, `attemptCount` reset to 0), sends it via notification-service (Section 4.4), returns 200 — "OTP sent to your email." **No stock check, no stock decrement, no order-status change happens here** — the cart is untouched until `verify` succeeds, so an abandoned OTP flow never leaves stock held hostage. |
| POST | `/api/orders/checkout/verify/{userId}` | Missing pending entry → 404. Expired → distinct error, `attemptCount` untouched. Wrong code → `attemptCount++`, 5th wrong guess → reject, require fresh `initiate`. Correct code → remove the pending entry and run **exactly** the existing v4 checkout logic, unchanged internally: validate stock for all items → decrement stock for all items → create order `PENDING_PAYMENT` → call payment-service → resolve email (already have it from `initiate`, but re-resolving is also fine if that's simpler to implement — either is correct) → on `SUCCESS`: `PAID`, send the (now upgraded, Section 5) bill-summary email → on `FAILED`: `PAYMENT_FAILED`, restock, send the failure email. Returns the placed order with its final status, same response shape as before. |
| POST | `/api/orders/checkout/resend/{userId}` | Missing pending entry → 404. Cooldown not elapsed → 429. Otherwise: new OTP, overwrite `otp`/`lastSentAt`/`expiresAt`, leave `attemptCount` untouched, resend, 200. |

This design deliberately does **not** restructure v4's checkout logic — it just wraps the existing atomic sequence behind an OTP gate, moving it from directly inside `POST /checkout/{userId}` to inside `POST /checkout/verify/{userId}`. Everything else about that sequence (stock validate-then-decrement ordering from v2, the `PENDING_PAYMENT`/`PAID`/`PAYMENT_FAILED` states from v4, the restock-on-failure compensation, the fail-open email lookup) is untouched.

### 4.4 New outbound call: order-service → notification-service (for the OTP itself)

**No new plumbing required.** order-service has called notification-service directly since v4, with its own `X-Internal-Secret`-attaching interceptor already in place (v4 Section 6.1). Sending the checkout OTP is just a new call site using the existing `NotificationServiceClient`:

- `message`: `"Your checkout OTP is {otp}. It expires in 5 minutes."`
- `orderId`: `null` — no order exists yet at `initiate` time, the `CART` order hasn't even been touched.
- `email`: resolved via the existing `GET /api/users/{userId}/email` call, same as always.

### 4.5 Ownership interceptor — path list update required

v5's ownership `HandlerInterceptor` (Section 2.4 of that file) is registered against a specific list of path patterns, one of which was `POST /api/orders/checkout/{userId}`. That exact path no longer exists — it must be updated to cover the three new paths instead: `/api/orders/checkout/initiate/{userId}`, `/api/orders/checkout/verify/{userId}`, `/api/orders/checkout/resend/{userId}`. The interceptor's logic itself (compare path `{userId}` against `X-User-Id`, admin bypass) doesn't change — only the registered pattern list does. Easy to miss since it's a one-line config change, not a code-logic change, but skipping it would silently reopen the exact ownership gap v5 closed.

---

## 5. notification-service — no contract change, richer content only

### 5.1 Decision: extend the message text, not the request shape

The request DTO stays exactly `{orderId, email, message}`, unchanged since v4. Rather than growing notification-service's contract to carry structured `items`/`total`/`status` fields, **order-service assembles the full bill-summary as a plain-text `message` string** before calling notification-service — same endpoint, same shape, just a longer, richer string than the old one-liner.

This is the simpler of the two options discussed: it needs zero changes to notification-service itself (still just logs + sends whatever `message` it's given via `JavaMailSender`, exactly as it has since v4), and it keeps the "does this replace or extend" question answered cleanly — **it replaces** the old one-liner entirely, same email, richer content, not a second email.

If you'd rather notification-service render its own structured template (e.g. an HTML table) from real `items`/`total` fields instead of a pre-built string, that's a valid alternative — say so and this section changes to grow the DTO instead. Defaulting to the text-assembly approach here since it touches fewer files.

### 5.2 Bill-summary message format (built by order-service, on `PAID`)

```
Your order #{orderId} has been placed successfully.

Items:
- {productNameSnapshot} x{quantity} — ${priceSnapshot} each
- {productNameSnapshot} x{quantity} — ${priceSnapshot} each
...

Total: ${orderTotal}
Status: PAID
```

Built from the same `OrderItem` snapshot fields (`productNameSnapshot`, `priceSnapshot`, `quantity`) that have existed since v1 — no new data is needed, order-service already has everything required to build this string at the point it's currently constructing the old one-line message.

The `PAYMENT_FAILED` message stays a short, single-line failure notice, unchanged from v4 — the bill-summary upgrade applies only to the success case, since there's no bill to summarize on a failed payment.

### 5.3 Two callers now, not one

notification-service gains a second caller (user-service, for registration OTPs, Section 3.4) alongside its existing caller (order-service, for checkout OTPs and the bill-summary/failure emails, since v4). No change is needed on notification-service's side to support this — its filter (checker) already authenticates by `X-Internal-Secret` alone, with no per-caller distinction, so a second legitimate sender needs nothing new.

---

## 6. api-gateway — whitelist and route updates

### 6.1 No new route entries

Both `user-route` (`Path=/api/users/**`) and `order-route` (`Path=/api/orders/**`) already match the new sub-paths by prefix — no new entry is appended to `spring.cloud.gateway.server.webmvc.routes`.

### 6.2 JWT filter whitelist — update, don't just append

The JWT filter's whitelist (v2 Section 2.2) currently names `POST /api/users/register` explicitly as one of the paths that bypasses validation. That exact path is retired — the whitelist entry must be updated to the three new paths instead:

- `POST /api/users/register/initiate`
- `POST /api/users/register/verify`
- `POST /api/users/register/resend`

All three stay public/unauthenticated, same reasoning as before — you can't attach a JWT for an account that doesn't exist yet.

`POST /api/orders/checkout/{userId}` was never on the whitelist (checkout has required a valid token since v2) — its three replacements (`initiate`, `verify`, `resend`, all under `/api/orders/checkout/**`) stay in the **authenticated** set, no whitelist change needed there beyond what Section 6.3 covers.

### 6.3 Internal-secret filter — no change

Unaffected, as always — it has no whitelist and checks every request uniformly, including the six new endpoints across both services.

---

## 7. frontend-service

| Page | Change | Calls |
|---|---|---|
| `register.html` | Unchanged fields (name, email, password, v6's role radio) — submit now calls `initiate` instead of the old single-step register | `POST /api/users/register/initiate` |
| `register-otp.html` (**new**) | A single OTP input field + submit button, plus a "Resend code" link/button. On a `429` from resend, redisplay this same page with a "please wait" message — no countdown, no JavaScript, consistent with this project's JS-free stance since v1 | `POST /api/users/register/verify`, `POST /api/users/register/resend` |
| `checkout.html` | Unchanged (still the v4 card-entry form, still cosmetic-only, Section 8.2 of v4) — submit now calls `initiate` instead of the old single-step checkout | `POST /api/orders/checkout/initiate/{userId}` |
| `checkout-otp.html` (**new**) | Same shape as `register-otp.html` — one OTP field, submit, resend link, server-rendered "please wait" on early resend | `POST /api/orders/checkout/verify/{userId}`, `POST /api/orders/checkout/resend/{userId}` |
| `order-confirmation.html` | Unchanged — still reflects `PAID`/`PAYMENT_FAILED` per v4, now just reached one step later in the flow (after `verify` succeeds) | none new |

No JavaScript is introduced anywhere in this version — every "please wait," "wrong code," and "OTP expired" message is a server-rendered page, exactly like every other validation error in this project since v1.

---

## 8. Known limitations (intentional, fixed in later versions or accepted permanently)

- **In-memory OTP state is lost on restart** — a pending registration or checkout in progress when user-service or order-service restarts simply disappears; the person has to start over from `initiate`. Acceptable given the explicit no-persistence call for this project's scale.
- **`initiate` always fully resets a pending flow**, with no cooldown of its own — someone could technically call `initiate` repeatedly to bypass the 30-second resend cooldown, since a fresh `initiate` isn't rate-limited the way `resend` is. Not worth closing for a project with no real abuse surface (localhost demo, ~10 users), but named explicitly rather than left implicit.
- **OTP is stored and compared as plain text, unhashed** — an accepted, deliberate choice (per your call) given its short lifetime and low value compared to a password.
- **No rate limiting beyond the 30-second/5-attempt rules** — no IP-based or account-based throttling on top of what's described here.
- **Checkout's stock check/decrement now happens later in the overall user journey** (at `verify`, not `initiate`) — between `initiate` and `verify`, another checkout could theoretically decrement stock first and leave the original person's `verify` call failing the stock check where it wouldn't have before OTP-gating existed. This is a minor, accepted timing gap, not a new architectural problem — it's the same "check-then-act" race that's existed since v2, just with a longer, human-paced window in the middle now.
- **Registration and checkout OTP delivery is not fail-open** — unlike the existing fail-open pattern for the bill-summary/failure email in the checkout success/failure path (v4), if notification-service can't be reached during `initiate` or `resend`, the person simply never receives a code and cannot proceed. This differs from v4's stance (where a failed notification never blocks checkout) because here the OTP *is* the gate, not a side effect — there's nothing to fail open to. Worth naming explicitly in your final write-up.

---

## 9. Testing checklist before calling v7 "done"

- [ ] `POST /api/users/register/initiate` with a brand-new email — returns 200, no `User` row created yet (confirm via DB), OTP email actually arrives
- [ ] `POST /api/users/register/initiate` with an email that already has a real `User` row — 409, same as v1's existing duplicate-email check
- [ ] `POST /api/users/register/verify` with the correct code — 201, `User` row now exists with the right `role` (confirm both `CUSTOMER` and `SELLER` still work correctly, re-run v6's `@PrePersist` checklist item here too)
- [ ] `POST /api/users/register/verify` with a wrong code — 400, `attemptCount` increments; repeat 5 times and confirm the 6th attempt (even with the correct code) is rejected and requires a fresh `initiate`
- [ ] Wait past 5 minutes without verifying, then attempt `verify` with the originally-correct code — confirm the distinct "expired" message, and confirm this did **not** count against the 5-attempt cap (check by then getting the count wrong twice and confirming you still have attempts left)
- [ ] `POST /api/users/register/resend` immediately after `initiate` (within 30s) — 429, "please wait"
- [ ] `POST /api/users/register/resend` after 30+ seconds — 200, new OTP arrives, old OTP no longer verifies, `attemptCount` unchanged from before the resend
- [ ] Log in with the newly-registered account (existing v6 login, no changes) — succeeds normally, `loginAs` role selector still works
- [ ] `POST /api/orders/checkout/initiate/{userId}` with an empty cart — 400, same as before
- [ ] `POST /api/orders/checkout/initiate/{userId}` with items in cart — 200, OTP arrives, confirm stock has **not** been decremented yet and the order has **not** been created yet
- [ ] `POST /api/orders/checkout/verify/{userId}` with the correct code — runs the full v4 checkout sequence correctly (stock decrements, payment call happens, order lands in `PAID` or `PAYMENT_FAILED` exactly as v4's existing behavior)
- [ ] Confirm the success-path email is now the full bill summary (items, quantities, prices, total, status) — not the old one-liner
- [ ] Confirm the failure-path email is still the short failure notice, unchanged from v4
- [ ] Repeat the wrong-code / attempt-cap / expiry / resend-cooldown checks from the registration checklist against the checkout flow
- [ ] Attempt `POST /api/orders/checkout/verify/{userId}` for a `userId` that doesn't match the caller's `X-User-Id` (CUSTOMER-role token) — confirm 403 from v5's ownership interceptor, proving Section 4.5's path-list update actually took effect
- [ ] Confirm the same mismatched-userId request succeeds for an ADMIN-role token, per v5's existing admin bypass
- [ ] Attempt `POST /api/users/register` (the old, retired path) directly — confirm it's gone (404 or no matching route), not silently still working alongside the new flow
- [ ] Attempt `POST /api/orders/checkout/{userId}` (the old, retired path) directly — same confirmation
- [ ] Temporarily stop notification-service, then attempt `initiate` for both registration and checkout — confirm the person genuinely cannot proceed without a code (this is the one place in the project that's deliberately not fail-open, per Section 8)
- [ ] Full frontend walkthrough, JS-free: register → land on OTP page → (optionally test the resend "please wait" message) → verify → log in → browse → cart → checkout → land on checkout-OTP page → verify → confirmation page shows the real bill summary