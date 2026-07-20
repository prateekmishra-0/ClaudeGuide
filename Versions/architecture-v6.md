# ARCHITECTURE — v6 (Seller Role)

> Carried over from v5: eureka-server, api-gateway, product-service, user-service, order-service,
> recommendation-service, frontend-service all unchanged from v5 except where explicitly called
> out below — see `services-index.md` for the summary. This file covers only what's new for
> sellers: self-service seller registration, a role-aware login flow, seller-scoped product
> ownership, and cascade-delete when a seller account is removed.
>
> Not in scope for this version: no OTP anywhere (that's v7), no seller-side sales dashboard or
> analytics, no seller approval workflow. Registration and login each stay single-step — this
> version only adds a role dimension to steps that already existed.

---

## 1. Services in this version

| Service | Port | Change from v5 |
|---|---|---|
| eureka-server | 8761 | unchanged |
| api-gateway | 8000 | unchanged — no new routes, no route changes (Section 6) |
| product-service | 8081 | `sellerId` added to `Product`; ownership enforcement on write endpoints; new internal-only bulk-delete-by-seller endpoint (Section 2) |
| user-service | 8082 | role selector on registration and login; new outbound Feign call to product-service; `@PrePersist` edit (Section 3) |
| order-service | 8083 | unchanged |
| recommendation-service | 8084 | unchanged |
| frontend-service | 8080 | registration form gets a role radio button; login form gets a role selector; minor product-detail/seller-panel additions (Section 5) |

No new services this version. No new databases, no new tables — `sellerId` is a new column on the existing `Product` table.

---

## 2. product-service

### 2.1 Entity change — Product

| Field | Type | Constraints |
|---|---|---|
| sellerId | Long | **new**, nullable at the column level (existing pre-v6 products have no seller), `@NotNull` is *not* applied at the entity level — enforced instead in the service layer only on create, and only for products created by a seller (Section 2.3) |

No migration needed for existing rows beyond the new nullable column — pre-v6 products simply have `sellerId = null` and are treated as admin/house-owned products going forward (any admin can still edit or delete them, no seller can claim them retroactively).

`sellerId` is **never** accepted as a request-body field on `POST /api/products` or `PUT /api/products/{id}` — if present in the JSON body it is silently ignored, not validated, not error'd on. The only way it's ever set is server-side, described in 2.3.

### 2.2 Where `sellerId` comes from

product-service does not talk to user-service and does not decode the JWT itself. Instead, api-gateway's existing JWT filter (since v2) is extended to forward two headers downstream on every authenticated request, not just `X-User-Id` (already forwarded since v2):

| Header | Source | Since |
|---|---|---|
| `X-User-Id` | JWT subject claim | v2 |
| `X-User-Role` | JWT `role` claim | **new in v6** |

This is the one gateway-level change in this version, and it's additive — existing consumers of `X-User-Id` are unaffected. product-service reads `X-User-Role` off the incoming request (via a request-scoped filter or simply `@RequestHeader`) to decide whether the caller is a `SELLER`, `CUSTOMER`, or `ADMIN`, and reads `X-User-Id` to know *which* seller.

### 2.3 Ownership rules on the existing endpoints

| Method | Path | Change |
|---|---|---|
| POST | `/api/products` | If `X-User-Role: SELLER` → `sellerId` is set from `X-User-Id` server-side, ignoring any `sellerId` in the body. If `X-User-Role: ADMIN` → `sellerId` stays `null` (house product) unless you explicitly decide otherwise later — not needed for this version. `CUSTOMER` role → 403, customers can't create products, same as always been implicitly true. |
| PUT | `/api/products/{id}` | Loads the existing product first. If `X-User-Role: SELLER`, compare `X-User-Id` against the product's stored `sellerId` — mismatch → 403 (`ForbiddenException`), even if the product's `sellerId` is `null` (a seller can never edit a house product). `ADMIN` bypasses this check entirely, can edit anything. |
| DELETE | `/api/products/{id}` | Same ownership check as `PUT`. `ADMIN` bypasses. |
| PUT | `/api/products/{id}/stock` | Same ownership check — a seller adjusting their own stock is legitimate, adjusting someone else's isn't. `ADMIN` bypasses. |
| GET | `/api/products`, `GET /api/products/{id}` | **Unchanged, no ownership check.** Browsing the catalog was never seller/customer-restricted and stays that way — a seller sees the exact same public catalog as anyone else, plus their own write access to their own listings. |

No new "my products" listing endpoint is introduced in this version — `GET /api/products?sellerId={id}` already works via the existing `categoryId`-style query-param filtering pattern from v1 extended to also accept `sellerId` (a small addition to the existing controller method, not a new endpoint), and the frontend's seller panel (Section 5) uses this with the logged-in seller's own ID.

### 2.4 New internal-only endpoint — cascade delete

| Method | Path | Purpose | Notes |
|---|---|---|---|
| DELETE | `/api/products/by-seller/{sellerId}` | hard-delete every product owned by this seller | **Internal-only. Never reachable through the gateway, not even by an ADMIN token.** This is a cascade side-effect of user-service deleting a user, not a standalone admin action. |

**Guard:** api-gateway's existing `product-route` (`Path=/api/products/**`) already matches this path, and the internal-secret filter alone doesn't distinguish "came through the gateway" from "came directly via Eureka." Close that gap the same way this project has closed it before: at the top of this controller method, check for the presence of `X-User-Id`. If it's present at all — meaning the request arrived through the gateway, from any caller — throw `ForbiddenException` → 403. If it's absent — a direct Eureka call, internal-secret-authenticated — proceed. This is the same pattern, so build it consistently if you're the one implementing it (rather than re-deriving it from scratch).

**Logic:**
1. Guard as above.
2. `long deletedCount = productRepository.deleteBySellerId(sellerId);` — one new derived-delete method on `ProductRepository`. Hard delete, consistent with this project's no-soft-delete stance since v1.
3. Return 200 with `{"deletedCount": N}` — informational, logged by the caller, not shown to any end user.

**Why this is safe to hard-delete:** `OrderItem` has snapshotted `productNameSnapshot` and `priceSnapshot` at add-to-cart time since v1, specifically so a product's later deletion never breaks past order history. Deleting a seller's whole catalog this way is no different from any individual product deletion that's always been allowed.

**No repository/entity change beyond the one new method and the one new `sellerId` column** — `ProductRequest`/`ProductResponse` gain `sellerId` as a response-only field (never accepted on input, per 2.1).

### 2.5 Internal-secret filter

No change — product-service's existing `OncePerRequestFilter` (since v2) already checks `X-Internal-Secret` on every request uniformly, no whitelist. The new `by-seller` endpoint is covered by this exactly like every other endpoint; the `X-User-Id` check in 2.4 is a *second*, independent guard layered on top, not a replacement for it.

---

## 3. user-service

### 3.1 Entity change — User

No new fields. The only change is behavioral, in `@PrePersist`:

**Before (v1–v5):**
```java
@PrePersist
public void prePersist() {
    this.role = "CUSTOMER";
    this.createdAt = LocalDateTime.now();
}
```

**After (v6):**
```java
@PrePersist
public void prePersist() {
    if (this.role == null) {
        this.role = "CUSTOMER";
    }
    this.createdAt = LocalDateTime.now();
}
```

This is the one edit to existing, previously-tested code in this version — everywhere else, v6 only adds new code. Without this change, a seller registering with `role = "SELLER"` set on the incoming entity would have it silently overwritten to `"CUSTOMER"` before the insert, which is exactly the kind of bug that's invisible until someone tries to log in as a seller and gets rejected by the role-mismatch check (3.3) for no apparent reason.

`role` still accepts only two values through the public registration endpoint — `CUSTOMER` and `SELLER` (3.2). `ADMIN` is still exclusively a manual-SQL, direct-database affair, exactly as in every prior version — there is no code path, endpoint, or request body field anywhere in this project that can produce an `ADMIN` row.

### 3.2 Registration — role selection

`POST /api/users/register` request body gains one new optional field:

| Field | Type | Notes |
|---|---|---|
| role | String | optional; if present, must be exactly `"CUSTOMER"` or `"SELLER"` (case-sensitive) — anything else, including `"ADMIN"`, is rejected with 400 via a small explicit check in the controller before the entity is even built, not left to `@PrePersist` to silently absorb. If omitted entirely, defaults to `CUSTOMER` via the `@PrePersist` change in 3.1. |

No approval step, no pending/verification state for sellers — a `SELLER` registration is a normal, immediate, self-service account creation, identical in every other respect (email uniqueness check, password hashing, 409 on duplicate email) to the existing v1 flow.

### 3.3 Login — role selector

`POST /api/users/login` request body gains one new required field:

| Field | Type | Notes |
|---|---|---|
| loginAs | String | **required**, one of `"CUSTOMER"`, `"SELLER"`, `"ADMIN"` |

New logic, inserted after the existing password check succeeds and before the token/JWT is issued:

1. Password check fails → **401**, exactly as before (v1). This check happens first, unchanged — a wrong password is still a wrong password regardless of what role was selected.
2. Password check succeeds → compare `loginAs` against the account's actual stored `role`.
    - Match → proceed to issue the token exactly as before, `role` claim embedded as it already is (since v2's JWT work).
    - Mismatch → **403**, a new `RoleMismatchException`, distinct from the existing 401 path, with a message along the lines of `"This account isn't registered as a {loginAs}."` — deliberately not the generic invalid-credentials message, since the credentials were in fact correct.

The selector is routing/UX only — it never grants access, and never changes which role ends up in the issued JWT. The JWT's `role` claim always comes from the database row, exactly as it always has since v2; `loginAs` only gates whether login is allowed to proceed at all, it's never trusted as the source of truth for anything downstream.

**New exception:** `RoleMismatchException` (package `com.ecommerce.userservice.exception`) → 403, wired into the existing `GlobalExceptionHandler` alongside the existing `InvalidCredentialsException` → 401.

### 3.4 New outbound call: user-service → product-service

This is user-service's **first-ever outbound inter-service call** in this project — every prior version had user-service called *by* others (product-service, order-service, recommendation-service all indirectly depend on user identity via the gateway's JWT, but nothing in user-service has ever called out to another service). This means some new plumbing that order-service and recommendation-service have had since much earlier versions is new territory here:

- `@EnableFeignClients` added to user-service's main application class — not previously needed since it had no Feign clients at all.
- New Feign client, `ProductServiceClient` (package `com.ecommerce.userservice.client`), `@FeignClient(name = "product-service")`.
    - One method: `@DeleteMapping("/api/products/by-seller/{sellerId}") void deleteProductsBySeller(@PathVariable("sellerId") Long sellerId)` — declarative, no request body.
- New `FeignInternalSecretInterceptor` (package `com.ecommerce.userservice.config`), a `RequestInterceptor` bean that attaches `X-Internal-Secret` to every outbound Feign call this service makes — mirrors the interceptor pattern order-service has used since v3/v4 for its own outbound calls. Registered once, applies automatically to `ProductServiceClient` (and any future Feign client user-service gains) with no per-call wiring.
    - Deliberately, this call carries **no** `X-User-Id` header — it's a direct Feign-to-Eureka call, not routed through the gateway, so there's nothing to strip; it just never had one to begin with. This is exactly what satisfies product-service's guard in Section 2.4.

**Configuration:** user-service's `application.yml` needs `internal.secret` set — it already has this value from prior versions (it's a filter *checker* for its own inbound endpoints since v2), so no new secret is introduced, the same literal string is now also used as a *sender* for the first time.

### 3.5 Change to the existing admin delete-user flow

`DELETE /api/users/{id}` (admin-only, existing since v5) gets one new step, inserted **before** the existing user-row deletion:

1. Load the user (existing lookup, unchanged).
2. **New:** if `user.getRole().equals("SELLER")`, call `productServiceClient.deleteProductsBySeller(id)`, wrapped in try/catch.
    - If this call fails for any reason (product-service unreachable, non-2xx response, etc.) → throw a new `SellerProductCleanupFailedException` → **502**, and **do not proceed to delete the `User` row.** This is intentionally **not** fail-open, unlike this project's usual pattern for non-critical side effects (recommendations failing empty, etc.) — an admin-requested deletion silently leaving orphaned products behind is a data-integrity problem, not a cosmetic one, so the whole operation aborts rather than half-completing.
3. If the role is `CUSTOMER` or `ADMIN`, skip step 2 entirely — no call is made, nothing to clean up.
4. If step 2 succeeded (or was skipped), proceed with the existing `User`-row deletion exactly as before, unchanged.

**New exception:** `SellerProductCleanupFailedException` (package `com.ecommerce.userservice.exception`) → 502, wired into the existing `GlobalExceptionHandler`, message along the lines of `"Could not remove this seller's products — user was not deleted."`

### 3.6 Internal-secret filter — no change

user-service's existing inbound filter (checker side, since v2) is unaffected — this version only adds a *sender* role to a service that previously only ever checked incoming requests.

---

## 4. order-service, recommendation-service, notification-service

No changes. Checkout, cart, recommendations, and (if you've already got it from elsewhere) notifications have never cared which "owner" a product has — a multi-seller cart just works with zero extra code, since every one of these services only ever deals with `productId`, never `sellerId`.

---

## 5. frontend-service

| Page | Change | Calls |
|---|---|---|
| `register.html` | Adds a role radio button: **Customer** / **Seller** (no Admin option, ever) | `POST /api/users/register` with the new optional `role` field |
| `login.html` | Adds a role selector: **Customer** / **Seller** / **Admin** | `POST /api/users/login` with the new required `loginAs` field. On 403 (role mismatch), show the specific "this account isn't registered as a {role}" message rather than the generic login-failed message used for 401. |
| `seller-products.html` (**new**) | Simple list/manage page for a logged-in seller's own products — reuses the existing product list template's layout, filtered via `GET /api/products?sellerId={loggedInUserId}` | product-service via gateway |
| `product-detail.html` | No structural change — a seller viewing their own product now also sees Edit/Delete links (shown only when the logged-in user's ID matches the product's `sellerId`, or the user is `ADMIN`) | none new |

`userId` and `role` for these checks come from the same `HttpSession`-backed mechanism established in v1/v2 — nothing new about how the frontend tracks who's logged in, just new UI branches based on the `role` already available there.

---

## 6. api-gateway

No new routes — `product-route` and `user-route` already cover everything this version touches, unchanged since their introduction in v2. The only gateway-level change is the one described in Section 2.2: the JWT filter now also forwards `X-User-Role` downstream, in addition to the `X-User-Id` it's forwarded since v2. This is additive to the filter's existing behavior, not a rewrite of it.

No whitelist changes — the write endpoints this version adds ownership checks to (`POST/PUT/DELETE /api/products/**`) were already in the authenticated set since v2; they don't need a *new* gate, they need the *existing* gate to also tell product-service *who* is authenticated, which is exactly what `X-User-Role` plus the already-forwarded `X-User-Id` provides.

The new `by-seller` endpoint is deliberately **not** added to any gateway route or whitelist — see Section 2.4, it's meant to be unreachable through the gateway entirely, guarded at the controller level instead.

---

## 7. Known limitations (intentional, fixed in later versions or accepted permanently)

- **No seller approval workflow** — anyone can self-register as a seller with no admin review. Accepted as a deliberate design choice for this version, not a gap to close later unless you decide otherwise.
- **No seller-side sales dashboard, analytics, or order visibility** — a seller can manage their own listings but has zero visibility into who bought what. Checkout, order history, and recommendations remain entirely seller-agnostic, per Section 4.
- **Pre-v6 products have `sellerId = null`** and are permanently un-claimable by any seller — only `ADMIN` can edit/delete them going forward, exactly as before this version existed.
- **Deleting a seller now hard-deletes all of their products** (Section 2.4, 3.4–3.5) — this is asymmetric with the still-accepted gap for *customers*: deleting a customer still leaves their past orders in place, snapshots intact, nothing cascades. That asymmetry is intentional — a customer's orders are historical records worth keeping; a seller's active listings are not something a deleted seller should keep controlling or benefiting from. If the cascade call to product-service fails, the user deletion itself is aborted rather than completing with orphaned products left behind — the admin sees a 502, not a silent partial success.
- **No OTP anywhere in this version** — registration and login are each still single-step, exactly as in v1–v5. OTP-gated registration/checkout is a separate, later addition (v7), layered on top of whatever shape registration/login are in once this version lands.
- **`X-User-Role` is trusted from the gateway with no independent re-verification inside product-service** — consistent with how `X-User-Id` has been trusted since v2 (the gateway is the sole JWT-verification point in this architecture by design); this is not a new trust boundary, just an extension of the existing one.

---

## 8. Testing checklist before calling v6 "done"

- [ ] Register a new account with `role: "SELLER"` — confirm the row in `user_db` actually has `role = "SELLER"`, not silently overwritten to `CUSTOMER"` (this is the `@PrePersist` change, Section 3.1 — test this one specifically, it's the easiest thing to get subtly wrong)
- [ ] Register a new account with `role: "CUSTOMER"`, and separately with `role` omitted entirely — both end up as `CUSTOMER`
- [ ] Register with `role: "ADMIN"` — confirm 400, no account created
- [ ] Log in as a seller with `loginAs: "SELLER"` — succeeds, JWT issued with `role: SELLER`
- [ ] Log in as that same seller account with `loginAs: "CUSTOMER"` — confirm 403 with the role-mismatch message, not 401, and confirm no token is issued
- [ ] Log in with a correct email but wrong password, any `loginAs` value — confirm 401 (unchanged from v1), not 403
- [ ] As a logged-in seller, create a product — confirm the created row's `sellerId` matches the seller's own user ID, even if you try to pass a different `sellerId` in the request body
- [ ] As that seller, edit and delete their own product — succeeds
- [ ] As that seller, attempt to edit or delete a *different* seller's product (or an admin-owned/pre-v6 product with `sellerId = null`) — confirm 403 in both cases
- [ ] As `ADMIN`, edit and delete any product regardless of `sellerId` — succeeds in every case
- [ ] As a `CUSTOMER`, attempt `POST /api/products` — confirm 403
- [ ] `GET /api/products` and `GET /api/products/{id}` — confirm identical, unrestricted behavior for customer, seller, and logged-out callers (no regression from v1)
- [ ] `GET /api/products?sellerId={id}` returns only that seller's products
- [ ] As `ADMIN`, delete a `SELLER` account that has at least 2 active products — confirm both the `User` row and all of that seller's `Product` rows are gone afterward
- [ ] Confirm any past order containing one of that seller's now-deleted products still displays correctly in order history (name/price come from `OrderItem`'s snapshot fields, unaffected by the product's deletion)
- [ ] Attempt `DELETE /api/products/by-seller/{sellerId}` directly through the gateway, with a valid ADMIN token — confirm 403, proving this endpoint is unreachable even by an admin except as user-service's internal cascade call
- [ ] Temporarily stop product-service, then attempt to delete a `SELLER` account as `ADMIN` — confirm the request fails with 502 and, critically, confirm the `User` row still exists afterward (the deletion did not partially complete)
- [ ] Delete a `CUSTOMER` account and a separate `ADMIN` account — confirm neither triggers any call to product-service at all (check product-service's logs show no incoming request for either case)
- [ ] Full existing v1–v5 walkthroughs (registration, login, browse, cart, checkout, order history, recommendations, existing admin panel flows) still pass unchanged