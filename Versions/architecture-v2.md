# ARCHITECTURE — v2 (Gateway + Real Auth + Inventory)

> Carried over from v1: eureka-server, product-service, user-service, order-service, frontend-service all still exist and keep their v1 responsibilities. This file only covers what changes or is added in v2. For full v1 entity/endpoint detail, see `services-index.md` for the summary or the (deleted-locally, still in git history) `architecture-v1.md` for the deep dive.
> New in v2: api-gateway (new service), JWT auth (user-service upgrade), stock decrement at checkout (product-service + order-service upgrade).

---

## 1. Services in this version

| Service | Port | Change from v1 |
|---|---|---|
| eureka-server | 8761 | unchanged |
| api-gateway | 8000 | **new** |
| product-service | 8081 | `/stock` endpoint now actually used; no entity changes; gains the internal-secret filter (Section 2.3) |
| user-service | 8082 | login now issues JWT instead of an in-memory token; User entity unchanged |
| order-service | 8083 | checkout flow now calls product-service's stock endpoint; no entity changes |
| frontend-service | 8080 | now calls everything through the gateway (port 8000) instead of resolving each service by name directly |

No new entities in v2. This version is entirely about the request path and auth mechanism, not new data.

---

## 2. api-gateway

`spring-cloud-starter-gateway`, registered with Eureka. This becomes the single entry point — frontend-service talks only to `localhost:8000` from now on; nothing else is called directly.

### 2.1 Route table

| Route ID | Predicate | Destination | Notes |
|---|---|---|---|
| product-route | `Path=/api/products/**,/api/categories/**` | `lb://product-service` | `lb://` = resolved via Eureka, load-balanced. GET requests on this path are public (see 2.2) — browsing the catalog must not require login, carrying forward v1's anonymous-browsing behavior. Any non-GET method on this same path (POST/PUT/DELETE, still admin-only in practice) requires a valid token. |
| user-route | `Path=/api/users/**` | `lb://user-service` | `/register` and `/login` on this path must bypass the JWT filter (see 2.2) — everything else on it requires a valid token |
| order-route | `Path=/api/orders/**` | `lb://order-service` | requires a valid token |

### 2.2 JWT validation filter

A custom `GlobalFilter` (Spring Cloud Gateway's filter type, not a Servlet `Filter` like v1's pattern would suggest — Gateway runs on a reactive stack) that:

1. Skips validation entirely for: `POST /api/users/register`, `POST /api/users/login`, and any GET request matching `/api/products/**` or `/api/categories/**` (whitelist by exact path + method, same idea as the Swagger whitelist pattern in your reference guide's `SecretKeyFilter`). The product/category GET whitelist is not optional — v1's frontend serves the homepage and product-detail pages to logged-out visitors, and v3's recommendation widget on product-detail also depends on anonymous access continuing to work; gating these behind a token would silently break both.
2. For everything else (all order-service paths, all non-GET product/category requests, all user-service paths except register/login), reads the `Authorization: Bearer <token>` header
3. Validates the JWT signature using the shared HMAC secret (see 3.2)
4. On missing/invalid/expired token → short-circuits the chain, returns 401 directly from the gateway, request never reaches the downstream service
5. On valid token → forwards the request unchanged downstream (downstream services do **not** re-validate the JWT in v2 — that's a deliberate simplification, see Known Limitations)

No entity, no database — this filter is pure in-memory logic per request.

---

### 2.3 Internal-secret filter — service-to-service trust

The JWT filter (2.2) proves a *user* is authenticated. It does nothing to stop someone from bypassing the gateway entirely and hitting a backend service's port directly (e.g. `curl localhost:8083/api/orders/checkout/1` with no token at all) — nothing downstream currently checks where a request actually came from. This section closes that gap with a second, independent mechanism: a fixed, shared secret string, unrelated to the JWT, that proves a request passed through api-gateway (or another trusted internal caller — see below), regardless of whether that request also carries a valid JWT.

**The two mechanisms are not interchangeable and both stay:**

| | JWT (2.2) | Internal secret (this section) |
|---|---|---|
| Proves | which user is calling | that the caller is a trusted internal service, not an external direct hit |
| Value | changes per user/session, expires | one fixed string, same on every request, no expiry |
| Checked by | api-gateway only | every backend service, on every endpoint, no exceptions — including `/register` and `/login`, which are JWT-whitelisted but NOT internal-secret-whitelisted |
| Attached by | nothing upstream (the browser sends nothing; the gateway is where it's first validated) | any service making an outbound call to another backend service |

**Who attaches it, precisely — this is not "gateway-only":** every service that calls another backend service directly must attach `X-Internal-Secret` on that outbound call, not just api-gateway. In v2 this means:
- api-gateway attaches it on every request it forwards downstream (to product-service, user-service, order-service)
- order-service attaches it on its own outbound Feign calls to product-service (`getProduct`, `adjustStock`) — this traffic never passes through the gateway at all, so if order-service doesn't attach it, product-service's filter will reject order-service's own calls with 401

This pattern repeats for every new inter-service call introduced in later versions — v3's recommendation-service calling order-service/product-service, and v4's order-service calling payment-service/notification-service, each need the same attach-on-outbound treatment, documented in their own architecture files.

**Who checks it:** product-service, user-service, and order-service each get a new servlet filter — a plain `OncePerRequestFilter`, registered at the highest precedence (`@Order(1)` or equivalent), running before the dispatcher servlet. It reads `X-Internal-Secret`, compares it via string equality against a value injected from `application.yaml` (key: `internal.secret`), and if missing or mismatched, writes a 401 directly and does not call `filterChain.doFilter()` — the request never reaches any controller. This check applies to every endpoint in the service, including the JWT-whitelisted register/login/public-GET routes — a public endpoint being reachable without a *user* token doesn't mean it should be reachable without going through the gateway at all.

api-gateway itself does not need this filter (nothing calls it as a downstream target), and eureka-server does not need it (no business endpoints, never called as an API target by anything).

**Configuration:** the same literal string value goes into `application.yaml`'s `internal.secret` key in api-gateway, product-service, user-service, and order-service — generated once, pasted identically into each service's prompt, same discipline already used for the JWT HMAC secret. This is a separate value from the JWT secret; the two must not be the same string.

**What this does and doesn't fix:** this closes direct-bypass access (hitting a service's port without going through the gateway at all) and, as a side effect, closes `X-User-Id` header-spoofing (an attacker can't inject that header if they can't get past the secret check to reach a controller in the first place). It does **not** address the separate userId-vs-JWT ownership gap named in Section 7 below — that's still about a legitimately-authenticated user acting on someone else's resource, a different problem this mechanism doesn't touch.

## 3. user-service changes

### 3.1 Login behavior change

`POST /api/users/login` — same request body as v1 (email, password). Response changes from `{"token": "<uuid>"}` to a signed JWT string. The in-memory `ConcurrentHashMap` session store from v1 is deleted entirely — no longer needed, since a JWT is self-contained and doesn't require server-side lookup.

### 3.2 JWT claims and signing

| Claim | Source | Purpose |
|---|---|---|
| `sub` | `user.id` | standard "subject" claim, used as the user identifier |
| `email` | `user.email` | convenience, avoids a lookup for display purposes |
| `role` | `user.role` | present now, not actually checked/enforced by anything yet in v2 — role-based authorization is a later hardening step, not a v2 requirement |
| `exp` | now + 2 hours | short-lived on purpose; there is no refresh-token flow in v2, re-login is the only path back once expired — acceptable for a demo project, worth stating as a known simplification |

Signed with HMAC-SHA256 using a shared secret. The secret is read from `application.yml`/environment variable in both api-gateway and user-service — same value in both, since the gateway validates tokens that user-service issued. (This shared-secret approach is why you don't need an asymmetric key pair here; that would be the production-grade choice but is unnecessary complexity for two internal services holding the same file.)

### 3.3 `GET /api/users/me` behavior change

No longer looks up a token in a map — it now reads the JWT claims already validated by the gateway (forwarded as headers, e.g. `X-User-Id`, `X-User-Email`, injected by the gateway filter after validation, so user-service doesn't need to re-parse the JWT itself). This is the tradeoff mentioned in 2.2's last point: user-service trusts the gateway's word rather than re-verifying.

---

## 4. product-service changes

### 4.1 `PATCH /api/products/{id}/stock` — now load-bearing

Existed as a stub in v1, now actually called. Behavior:
- Request body: `{"delta": -2}` (negative to decrement, positive to restock)
- Guard: reject if `stockQuantity + delta < 0` → return 409 Conflict with a clear message, do **not** allow stock to go negative
- On success: apply the delta, return the updated product

This single endpoint is what both the "decrement on checkout" and "restock on payment failure" flows (checkout now, payment-failure restock lands in v4) will call — same endpoint, opposite-sign delta.

---

## 5. order-service changes

### 5.1 Checkout flow — now three steps instead of one

v1's checkout just flipped `CART` → `PLACED`. v2 adds validation and stock mutation before that flip:

```
POST /api/orders/checkout/{userId}
  1. Load the CART order and its items
  2. For EVERY item: call product-service GET /api/products/{id}, confirm stockQuantity >= requested quantity
     — check ALL items before mutating ANYTHING (see note below)
  3. Only if all items pass: for EVERY item, call product-service PATCH /api/products/{id}/stock with delta = -quantity
  4. Flip order status CART → PLACED
  5. Return the placed order
```

**Why validate all items before decrementing any:** if you decrement item 1's stock and then discover item 3 is out of stock, you'd need to undo item 1's decrement — a compensating action you'd rather avoid in v2. Checking everything first, mutating only after all checks pass, sidesteps that problem entirely for now. (Real partial-failure-mid-mutation handling, e.g. if the server crashes between step 3's individual calls, is out of scope until v4's payment-failure-restock logic sets the pattern for compensation.)

If any item fails the stock check in step 2 → checkout aborts, returns 409, cart is left untouched (still `CART` status, nothing was decremented).

---

## 6. frontend-service changes

- All outbound calls (`RestClient` base URL) now point at `http://api-gateway/...` (resolved via Eureka, same `lb://` mechanism) instead of individual service names.
- Every authenticated call now attaches `Authorization: Bearer <token>` — the JWT obtained at login, stored in the frontend's own `HttpSession` (replacing v1's opaque session token in that same session slot — the storage location doesn't change, only what's stored there).
- Checkout page should now surface the 409 "insufficient stock" case with a real user-facing message instead of a generic error page — this is the one UI change v2 actually requires.
- frontend-service does not attach or check `X-Internal-Secret` — it only ever calls api-gateway (never a backend service directly), and the gateway is the trust boundary between the browser-facing tier and the internal service mesh. This filter is an internal-only mechanism.

---

## 7. Known limitations (intentional, fixed in later versions)

- Downstream services trust the gateway's JWT validation instead of re-checking themselves — acceptable for a single-gateway setup where nothing bypasses it, but worth naming explicitly as a chosen simplification, not an oversight
- No refresh tokens — 2-hour hard expiry, re-login only
- No role enforcement despite the `role` claim existing — anyone with a valid token can call any authenticated endpoint
- No circuit breaker if product-service or user-service is down — gateway calls just fail through with a raw error (v5)
- No compensation logic if the server crashes mid-way through decrementing multiple items in one checkout (v4 introduces the pattern via payment-failure restock; a mid-decrement crash specifically is a known gap even after v4, worth naming in your final "known limitations" writeup)
- `userId` taken from the path variable (e.g. `/api/orders/checkout/{userId}`, `/api/orders/cart/{userId}/...`) is never cross-checked against the authenticated identity in the JWT (`sub` claim, forwarded as `X-User-Id`) — a logged-in user could act on another user's cart/order/history by changing the path id. This is a stated gap, not fixed until role/ownership enforcement is added; acceptable for a demo project with no real user data at stake, but worth naming explicitly rather than leaving unstated.
- The internal-secret mechanism (Section 2.3) is a single static, unrotatable string shared across four services — if it leaks (checked into a public repo, logged, etc.), anything holding it can forge internal service traffic indefinitely, with no expiry and no per-caller granularity. This is a deliberately lightweight trade-off (equivalent to a shared API key) rather than mutual TLS or per-request signing, acceptable for a localhost demo project where the backend ports aren't reachable from outside your own machine anyway.

---

## 8. Testing checklist before calling v2 "done"

- [ ] api-gateway dashboard/logs show routes registered for all three backend services
- [ ] `/api/users/register` and `/api/users/login` work with no token required
- [ ] Any authenticated endpoint (order-service paths, non-GET product/category requests, `/api/users/me`) without a token returns 401 from the gateway (check logs — request should never reach the downstream service)
- [ ] GET `/api/products`, `GET /api/products/{id}`, and `GET /api/categories` all work with NO token present — confirm the homepage and product-detail pages still render correctly for a logged-out visitor through the gateway
- [ ] Login returns a JWT; decode it manually (jwt.io or similar) and confirm `sub`, `email`, `role`, `exp` are present and correct
- [ ] Checkout with sufficient stock: stock visibly decreases in product-service afterward
- [ ] Checkout with insufficient stock: returns 409, cart remains `CART`, stock unchanged
- [ ] Full frontend walkthrough (register → login → browse → cart → checkout) works end to end through the gateway with zero direct service calls bypassing it
- [ ] Attempt to hit product-service, user-service, or order-service directly on its own port (bypassing api-gateway entirely) with no `X-Internal-Secret` header — confirm 401 from the service's own filter, request never reaches a controller
- [ ] Confirm the same direct-hit attempt still fails even against `/register` or `/login` — proving the internal-secret check has no whitelist, unlike the JWT filter
- [ ] Confirm order-service's checkout flow still succeeds end-to-end (through the gateway) — proving order-service correctly attaches the secret on its own outbound calls to product-service, not just relying on the gateway's already-attached header from the original inbound request
