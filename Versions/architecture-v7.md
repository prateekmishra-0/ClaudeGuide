# ARCHITECTURE — v7 (Seller Role)

> Carried over from v6: all 8 services exist and are unchanged in shape unless stated otherwise below — see `services-index.md` for the summary. No new services in v7. This version adds a third role, `SELLER`, that can manage only the products it created, and explicitly never sees any user/order data — a strict subset of `ADMIN`'s access, not an extension of it.
> **Services with zero changes in this version:** order-service, payment-service, notification-service, recommendation-service, eureka-server, api-gateway (no new routes, no whitelist changes). Checkout, cart, and order history remain completely seller-unaware — a multi-seller order works today with zero extra code, since `OrderItem.productId` was always just a plain reference, never tied to who manages that product.

---

## 1. What's added (by service)

| Service | Addition |
|---|---|
| product-service | New `sellerId` field on `Product`; write endpoints now accept `SELLER` in addition to `ADMIN`, gated by per-resource ownership for update/delete/stock (Section 2) |
| user-service | Registration gains an optional account-type choice (`CUSTOMER` or `SELLER`); one small, explicitly-flagged change to `User`'s existing `@PrePersist` logic (Section 3) |
| frontend-service | Register form gains a role choice; new seller panel at `/seller/**`, structurally parallel to but separate from the admin panel (Section 4) |

---

## 2. product-service — ownership-scoped product writes

### 2.1 Entity change: `Product` gains one field

| Field | Type | Constraints |
|---|---|---|
| sellerId | Long | nullable, `@Column(name = "seller_id")` — plain reference to a `user-service` id, no `@ManyToOne`, same "raw FK column" convention as `categoryId` since v1. **Null means platform/admin-owned** — every product created before v7, and every product an ADMIN creates from here on, has `sellerId = null`. Non-null means a specific seller owns it. |

No other field changes. No new table. `ddl-auto: update` adds this as a nullable column; existing rows get `null` automatically.

### 2.2 DTO changes

- `ProductRequest` (used by `POST`/`PUT /api/products`): **unchanged, no `sellerId` field added.** This is deliberate — `sellerId` is never accepted as client input, on any request, from anyone. It is derived server-side from `X-User-Id`/`X-User-Role`, never trusted from a request body. This closes off the obvious privilege-escalation vector (a seller setting `sellerId` to someone else's id, or an ADMIN request accidentally assigning ownership) before it can exist.
- `ProductResponse`: gains `sellerId` (nullable Long) — additive field, so the seller panel's own product list can be built from the same response shape everything else already uses. This value is a plain numeric id, not seller PII (name/email) — it is not a violation of "sellers see no user data," since no endpoint anywhere in this version resolves that id to a name, email, or any other profile detail for a seller. It is genuinely just a foreign-key-shaped tag, already effectively public the same way `categoryId` has been since v1.

### 2.3 Authorization — splitting the existing admin-only gate

v5 introduced one `AdminOnlyInterceptor` covering all five product-service write endpoints (`POST /api/categories`, `POST /api/products`, `PUT /api/products/{id}`, `DELETE /api/products/{id}`, `PUT /api/products/{id}/stock`), requiring `ADMIN` uniformly. v7 needs two different rules for two different groups of those endpoints, so the registration is split — **`AdminOnlyInterceptor` itself is not modified**, only which paths it's registered against:

| Endpoint group | Required role(s) | Handled by |
|---|---|---|
| `POST /api/categories` | `ADMIN` only — unchanged from v5 | existing `AdminOnlyInterceptor`, **narrowed** to just this one path in the `WebMvcConfigurer` registration |
| `POST /api/products`, `PUT /api/products/{id}`, `DELETE /api/products/{id}`, `PUT /api/products/{id}/stock` | `ADMIN` or `SELLER` | **new** `SellerOrAdminInterceptor` (package `com.ecommerce.productservice.security`), registered against these four paths |

`SellerOrAdminInterceptor`'s `preHandle` reads `X-User-Role`; if it's neither exactly `"ADMIN"` nor exactly `"SELLER"`, throw the existing `ForbiddenException` (local to product-service, from v5) → 403, same message shape as before. This is a coarse role gate only — same shape and scope as every interceptor in this project since v2, no database access inside it.

### 2.4 Authorization — per-resource ownership (new, in the controller/service layer)

The coarse role gate above only answers "is this caller ADMIN or SELLER at all" — it can't know whether a given `SELLER` owns a given `{id}` without loading that row, and this project's interceptors have never done database lookups. So the ownership check lives where every other resource-lookup already lives: inside `ProductController` (or `ProductService`, if this codebase already separated that out — follow whatever it already does, per `promptrule.md`'s "match the existing pattern" rule).

**New exception** (package `com.ecommerce.productservice.exception`): `NotProductOwnerException` → 403, `{"message": "You do not own this product"}`, wired into the existing `GlobalExceptionHandler` exactly like every other custom exception.

**Logic, applied identically to `PUT /api/products/{id}`, `DELETE /api/products/{id}`, and `PUT /api/products/{id}/stock`:**
1. Load the product (existing `findById` → `ProductNotFoundException` if missing, unchanged).
2. Read `X-User-Role` and `X-User-Id` (new — these three write-adjacent methods now read `X-User-Id` from the request for the first time in this service's history; every earlier version only ever needed `X-User-Role`).
3. If role is `"ADMIN"` → proceed unconditionally, exactly as v5 always allowed (admin retains full authority over every product, seller-owned or not — this is intentional, admin is the platform owner, not just "a bigger seller").
4. If role is `"SELLER"` → compare `X-User-Id` (parsed as `Long`) to `product.getSellerId()`. Since `X-User-Id` is always non-null (guaranteed by the gateway) and `sellerId` may be `null`, write the comparison as `userId.equals(product.getSellerId())` — **not** the reverse — so a `null` `sellerId` (an admin-owned or pre-v7 product) naturally and correctly fails the check with no special-case `null` handling needed. On mismatch → throw `NotProductOwnerException`.

**Logic for `POST /api/products` (create):**
- If role is `"SELLER"` → set `product.setSellerId(X-User-Id)` before saving, unconditionally — a seller can never create a product for anyone but themselves, and there's no field in `ProductRequest` for them to attempt otherwise.
- If role is `"ADMIN"` → `sellerId` stays `null`, unchanged from every version before v7.

### 2.5 New optional filter on the existing product list

`GET /api/products` (existing, paginated, since v1) gains one more optional query parameter, `?sellerId=`, alongside the existing `?categoryId=`. When present, filters to that seller's products only — powers the seller panel's "my products" list (Section 4.4). New repository method: `Page<Product> findBySellerId(Long sellerId, Pageable pageable)`. **Not supported in this version:** combining `categoryId` and `sellerId` filters in the same request — pick one or the other; this endpoint has never needed to combine filters before and doesn't start now. This remains a fully public, unauthenticated-or-authenticated GET, unchanged in every other respect — anyone (including a customer browsing normally) could technically pass `?sellerId=` too; that's harmless, same reasoning as Section 2.2's note on `sellerId` not being sensitive.

### 2.6 What this does not touch

`CategoryController` is completely unchanged — categories remain ADMIN-only to create, full stop; sellers choose from the existing list when creating a product, they cannot add to it. `GET /api/products/{id}`, `GET /api/categories` are unchanged. `InsufficientStockException`, `ProductNotFoundException`, `CategoryNotFoundException`, `CategoryAlreadyExistsException` are all unchanged.

---

## 3. user-service — becoming a seller at registration

### 3.1 Design decision, stated explicitly

Unlike `ADMIN` (v5: manual SQL promotion only, deliberately not self-service, since admin is a trusted-operator role), `SELLER` **is** self-service at registration — a normal signup choice, the same way a marketplace lets anyone start selling. This is a real product decision, not an oversight: there is no approval step, no vetting. Named again in Known Limitations (Section 6).

### 3.2 DTO change: `RegisterRequest` gains one field

- `accountType`: String, optional, no `@NotBlank` (absence is valid and means customer). Not an enum type — this project has consistently kept role-like fields as plain validated strings (`Order.status` since v1, `User.role` since v1) rather than introducing a Java enum, and this follows the same convention.

### 3.3 Server-side normalization — the actual security boundary

In `UserService`'s existing `initiateRegistration` method (v6), the `accountType` value is normalized, **not trusted as-is**: if it equals exactly `"SELLER"`, the resolved role is `"SELLER"`; for **any other value at all** — including blank, missing, `"CUSTOMER"`, or someone attempting `"ADMIN"` — the resolved role is `"CUSTOMER"`. This is written as an allow-list of one value, not a deny-list, specifically so that sending `"ADMIN"` (or anything else) through this field can never produce anything other than `"CUSTOMER"`. This is the only place in the whole registration flow role selection is decided — nowhere else reads `accountType` again.

### 3.4 Entity change: `PendingRegistration` (v6) gains one field

| Field | Type | Constraints |
|---|---|---|
| role | String | `@NotBlank`, `@Column(length = 20)` — the normalized value from Section 3.3 (`"CUSTOMER"` or `"SELLER"`), set once at initiate time, carried through to `verify` |

### 3.5 Change to the existing `/register/verify` logic (v6)

v6's verify step said: *"create the real `User` row (role defaults `'CUSTOMER'` via the existing `@PrePersist`, unchanged since v1)."* That line changes: the `User` is now constructed with `role` set explicitly to `pendingRegistration.getRole()` **before** persisting, rather than leaving it to the entity's default.

### 3.6 Small, explicitly-flagged change to `User`'s `@PrePersist`

Since v1, `User`'s `@PrePersist` has unconditionally set `role = "CUSTOMER"`, because nothing before v7 ever constructed a `User` with a role already assigned (ADMIN promotion has always happened via manual SQL, *after* creation, per v5 — never through this code path). Section 3.5 now needs to construct a `User` with `role = "SELLER"` already set, so the hook must change from an unconditional assignment to a conditional one: **only default to `"CUSTOMER"` if `role` is not already set**, i.e. `if (this.role == null) { this.role = "CUSTOMER"; }`. This is the one place in this version that touches previously-tested code rather than adding new code — call it out explicitly in the fix/build prompt for this file, per `promptrule.md`'s consistency rules. It does not affect the existing manual-SQL ADMIN-promotion path at all, since that happens after creation either way.

### 3.7 What this does not touch

`POST /api/users/login`, `GET /api/users/me`, `GET /api/users/{id}/email` (v4), the admin user-management endpoints (v5 Section 3), and every OTP mechanic from v6 (expiry, attempt cap, resend cooldown) are all completely unchanged — `accountType`/`role` is carried alongside the existing OTP fields on `PendingRegistration`, not instead of them.

---

## 4. frontend-service — seller-facing registration and panel

### 4.1 `register.html` change

Adds a plain HTML choice — two radio buttons, "Register as: Customer / Seller" (or an equivalent `<select>`) — bound to a new `accountType` field on the existing registration form-backing object, defaulting to "Customer" if nothing is selected. No JavaScript, same constraint as every template since v1. `AuthController`'s existing `POST /register` handler (already modified once in v6 to call `initiateRegistration`) needs no further logic change beyond passing this one extra field through in the request it already builds — the OTP flow itself (v6) is completely unaware of and unaffected by this field.

### 4.2 New interceptor: `SellerRequiredInterceptor`

Package `com.ecommerce.frontendservice.config`. Same shape as v5's `AdminRequiredInterceptor`: `preHandle` checks `HttpSession` attribute `"userRole"` (already populated at login since v5, no change needed there — it can now legitimately hold `"SELLER"`). If not exactly `"SELLER"`, redirect to `/` and return `false` — same silent-bounce behavior as the admin interceptor, so the seller panel's existence isn't advertised either. Registered against path pattern `/seller/**`.

**Deliberately not shared with `/admin/**`:** an `ADMIN` account visiting `/seller/**` is also bounced home. The seller panel and the admin panel are kept strictly separate — admin already has full product authority via `/admin/**` (Section 2.4 confirms this at the API level too), so there's no need for admin to also use the seller UI.

**Security note, same as v5's:** this interceptor is UX-only. The real enforcement is product-service's `SellerOrAdminInterceptor` + per-resource ownership check (Section 2.3, 2.4), which check independently of anything the frontend does.

### 4.3 Feign client change

One new declarative method on the existing `ProductServiceClient`: `@GetMapping("/api/products") ProductPageResponse getProductsBySeller(@RequestParam("sellerId") Long sellerId)`, targeting Section 2.5's new query filter. No other new Feign methods are needed — sellers use the exact same create/update/delete/stock methods the admin panel already added in v5 Section 5.3; product-service's own authorization (Section 2) is what makes the same calls behave differently depending on who's calling.

### 4.4 New controller: `SellerProductController`

Package `com.ecommerce.frontendservice.controller`, base path `/seller`.

| Route | Calls | Purpose |
|---|---|---|
| `GET /seller/products` | `productServiceClient.getProductsBySeller(userId)` (`userId` from the existing session attribute, unchanged since v1) | list only this seller's own products, with real stock counts (extending v5 Section 4's stock-visibility rule: sellers see real numbers for their own listings, same reasoning as admin) |
| `GET /seller/products/new` | `GET /api/categories` (existing) | blank creation form, category dropdown — same categories admin uses, sellers cannot add new ones (Section 2.6) |
| `POST /seller/products/new` | `POST /api/products` (existing method) | create; the seller's identity is carried by the JWT (`FeignAuthInterceptor`, since v2) — this form never has a seller/owner field, since ownership is assigned server-side (Section 2.4) |
| `GET /seller/products/{id}/edit` | `GET /api/products/{id}` (existing, public) | prefilled edit form — note this GET succeeds even for a product the seller doesn't own, since the read endpoint has no ownership check; the write that follows is what's actually gated |
| `POST /seller/products/{id}/edit` | `PUT /api/products/{id}` | update; catch `FeignException.Forbidden` (product-service's new `NotProductOwnerException`) → flash an error, redirect to `/seller/products` — same "translate a Feign 4xx into a flash message" pattern already used for `InsufficientStockException`/`EmptyCartException` elsewhere in this project |
| `POST /seller/products/{id}/delete` | `DELETE /api/products/{id}` | delete; same `Forbidden` handling as above |
| `POST /seller/products/{id}/stock` | `PUT /api/products/{id}/stock` | stock adjustment; same `Forbidden` handling as above |

### 4.5 New templates

`seller-products.html`, `seller-product-form.html` (shared new/edit). Same plain-HTML-forms, no-JavaScript constraint as every template since v1. Header fragment gains one more conditional link — "Seller Dashboard" → `/seller/products`, shown only `th:if="${loggedInUserRole == 'SELLER'}"`, alongside the existing admin-panel and logged-in-user links.

### 4.6 What NOT to build in this version

- No seller-facing order/sales visibility of any kind — no "products sold," no buyer counts, no revenue. This is the direct implementation of "sellers see no user data": no template, controller, or Feign call in this section ever calls `OrderServiceClient` or `UserServiceClient`. (Contrast with Section 5.4a of `architecture-v5.md`, the admin-only "who bought this" view — that stays exactly where it is, admin-only, untouched by this version.)
- No seller attribution shown on customer-facing pages (`home.html`, `product-detail.html`) — "sold by X" is not part of this version's scope.
- No seller self-analytics (view counts, ranking, etc.).
- No product reassignment between sellers, or from a seller back to admin — `sellerId` is set once at creation and never exposed as an editable field anywhere.

---

## 5. api-gateway

No changes. The existing `product-route` (`Path=/api/products/**`) and `user-route` (`Path=/api/users/**`) already match every path this version touches. No whitelist changes — the product-write endpoints were already authenticated-only since v5 (a valid JWT required), and product-service's own interceptor (Section 2.3) is what now also accepts `SELLER` in addition to `ADMIN` on top of that same authentication requirement.

---

## 6. Known limitations (intentional, not solved here)

- **No seller approval/vetting workflow** — anyone can self-declare as `SELLER` at registration (Section 3.1). A real marketplace would gate this behind admin review; this project accepts the open version.
- **No seller-facing sales visibility at all** — by design, per this version's explicit "sees no user data" requirement. A future version could add a narrow, purchase-count-only view (no buyer identities) without contradicting that requirement, but it isn't built here.
- **No cascading behavior if a seller account is deleted** via the admin panel (v5 Section 3) — their products simply remain, with a `sellerId` that no longer resolves to a real user. Same shape of accepted gap as v5's user-deletion-vs-orders limitation, extended to products.
- **No combined `categoryId` + `sellerId` filtering** on `GET /api/products` (Section 2.5) — a minor query-flexibility gap, not a correctness issue.
- **`sellerId` is permanent once set** — no transfer/reassignment endpoint exists, for either role.

---

## 7. Testing checklist before calling v7 "done"

- [ ] Register a new account choosing "Seller" — confirm (after OTP verification, per v6) the created `User` row has `role = "SELLER"`, not `"CUSTOMER"`
- [ ] Register a new account choosing "Customer" (or leaving the choice blank) — confirm `role = "CUSTOMER"`, unchanged behavior from before v7
- [ ] Attempt to register by directly POSTing `accountType: "ADMIN"` to `/api/users/register` (bypassing the frontend form, e.g. via Postman) — confirm the created user ends up `"CUSTOMER"`, never `"ADMIN"` (Section 3.3)
- [ ] As a SELLER, create a product via `/seller/products/new` — confirm in the database that `sellerId` matches the seller's own user id
- [ ] As the same SELLER, edit and adjust stock on that product — confirm both succeed
- [ ] As a **different** SELLER (a second seller account), attempt to edit or delete the first seller's product by guessing/tampering the `{id}` in the URL — confirm 403 at product-service, and confirm the frontend shows a flash error and redirects to `/seller/products` rather than a raw error page
- [ ] As ADMIN, edit, adjust stock on, and delete a product that belongs to a specific SELLER — confirm all three succeed (admin's full authority, Section 2.4, is unaffected by seller ownership)
- [ ] As ADMIN, create a new product — confirm its `sellerId` is `null`, and confirm a SELLER account