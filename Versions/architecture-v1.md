# ARCHITECTURE — v1 (Bedrock)

> Scope: the minimum working slice — register, browse, cart, checkout, order history.
> No gateway, no JWT, no recommendations, no payment, no notifications. Those are v2–v5.
> This file covers only what's built in v1. For services from other versions, see `services-index.md`.

---

## 1. Services in this version

| Service | Port | Datastore | Eureka client? |
|---|---|---|---|
| eureka-server | 8761 | none | n/a (is the registry) |
| product-service | 8081 | Postgres, schema `product_schema` | yes |
| user-service | 8082 | Postgres, schema `user_schema` | yes |
| order-service | 8083 | Postgres, schema `order_schema` | yes |
| frontend-service | 8080 | none (calls the above via RestClient) | yes |

All four Spring Boot apps (everything except eureka-server) register with Eureka on startup using `spring-cloud-starter-netflix-eureka-client`. No API Gateway yet — frontend-service resolves the other three by Eureka service name directly.

---

## 2. eureka-server

Plain `spring-cloud-starter-netflix-eureka-server` app. No entities, no REST endpoints of its own beyond the built-in dashboard. `application.yml` sets `register-with-eureka: false` and `fetch-registry: false` since it's the registry, not a client.

---

## 3. product-service

### 3.1 Entities

**Category**

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-generated (IDENTITY) |
| name | String | `@NotBlank`, `@Column(unique = true)`, max 60 |
| description | String | max 255, nullable |

**Product**

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-generated (IDENTITY) |
| name | String | `@NotBlank`, max 100 |
| description | String | max 500, nullable |
| price | BigDecimal | `@NotNull`, `@DecimalMin("0.01")` |
| stockQuantity | Integer | `@NotNull`, `@Min(0)` |
| categoryId | Long | `@NotNull` — plain FK column, no `@ManyToOne` object mapping needed in v1 (keep it a raw ID; avoid JPA relationship complexity until it's actually needed) |
| createdAt | LocalDateTime | set once in a `@PrePersist` method, not client-settable |

7 fields on Product, 3 on Category. No soft delete in v1 — deletes are hard deletes. (Soft delete is a legitimate pattern, see the reference guide you shared, but it's not needed until there's real data you'd regret losing — treat it as a v5-tier hardening candidate if you want it later.)

### 3.2 Endpoints

| Method | Path | Purpose | Notes |
|---|---|---|---|
| GET | `/api/categories` | list all categories | |
| POST | `/api/categories` | create category | admin use, no auth check yet in v1 |
| GET | `/api/products` | list products | supports `?categoryId=` filter, plain pagination via `Pageable` |
| GET | `/api/products/{id}` | get one product | 404 if missing |
| POST | `/api/products` | create product | admin use |
| PUT | `/api/products/{id}` | update product | full replace |
| DELETE | `/api/products/{id}` | hard delete | |
| PATCH | `/api/products/{id}/stock` | adjust stock | body: `{"delta": -2}` — used by order-service at checkout in v2+; not called by anything yet in v1, but build it now so v2's inventory work doesn't require a schema change |

### 3.3 Validation / error behavior

Standard `@RestControllerAdvice` `GlobalExceptionHandler`: catches `MethodArgumentNotValidException` → 400 with field errors, catches a custom `ProductNotFoundException` → 404, generic `Exception` fallback → 500. No need for anything more elaborate in v1.

---

## 4. user-service

### 4.1 Entities

**User**

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-generated (IDENTITY) |
| name | String | `@NotBlank`, max 60 |
| email | String | `@NotBlank`, `@Email`, `@Column(unique = true)` |
| passwordHash | String | `@NotBlank` — store BCrypt hash, never plaintext, use `spring-security-crypto`'s `BCryptPasswordEncoder` even though full Spring Security isn't wired in yet |
| role | String | default `"CUSTOMER"` in `@PrePersist`; no other roles actually enforced anywhere in v1, field exists so v2's role-based JWT claims don't require a schema change |
| createdAt | LocalDateTime | set in `@PrePersist` |

5 fields. No separate Session entity — session tokens are held in an **in-memory** `ConcurrentHashMap<String token, Long userId>` inside the service (not persisted to Postgres). This is a deliberate v1 limitation: restarting user-service logs everyone out. Acceptable now, fixed implicitly once v2 replaces this whole mechanism with JWT (which needs no server-side session store at all).

### 4.2 Endpoints

| Method | Path | Purpose | Notes |
|---|---|---|---|
| POST | `/api/users/register` | create account | body: name, email, password (plaintext in, hashed before save); 409 if email exists |
| POST | `/api/users/login` | authenticate | body: email, password; returns `{"token": "..."}` on success, 401 on bad credentials |
| GET | `/api/users/me` | resolve current user | header `X-Session-Token`; 401 if token missing/unknown; returns id, name, email, role |

---

## 5. order-service

Cart and order are the same entity in v1, distinguished by `status`. Splitting them into separate services is not a v1 (or even v2) concern.

### 5.1 Entities

**Order**

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-generated (IDENTITY) |
| userId | Long | `@NotNull` — plain reference, no cross-service FK (different DB schema) |
| status | String | `@NotNull` — one of `CART`, `PLACED` (plain String field in v1, not a full enum type in Postgres — keep the DB simple, validate the value in Java) |
| createdAt | LocalDateTime | `@PrePersist` |
| updatedAt | LocalDateTime | `@PreUpdate` |

**OrderItem**

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-generated (IDENTITY) |
| orderId | Long | `@ManyToOne` to Order, `@JoinColumn`, not nullable |
| productId | Long | `@NotNull` — reference into product-service, not a JPA relationship |
| productNameSnapshot | String | max 100 — copied from product-service at time of add-to-cart, so the cart/order still reads correctly even if the product is later renamed or deleted |
| priceSnapshot | BigDecimal | `@NotNull` — copied at checkout time, not read live, so a price change after checkout doesn't retroactively alter a placed order |
| quantity | Integer | `@NotNull`, `@Min(1)` |

5 fields on Order, 6 on OrderItem.

### 5.2 Endpoints

| Method | Path | Purpose | Notes |
|---|---|---|---|
| GET | `/api/orders/cart/{userId}` | view current cart | finds or implicitly treats "no CART order" as an empty cart |
| POST | `/api/orders/cart/{userId}/items` | add item to cart | body: productId, quantity. Calls product-service (`GET /api/products/{id}`) to fetch name+price for the snapshot fields and to confirm the product exists; 404 if product-service says it doesn't |
| DELETE | `/api/orders/cart/{userId}/items/{itemId}` | remove item from cart | |
| POST | `/api/orders/checkout/{userId}` | finalize the cart | transitions the CART order to PLACED; 400 if cart is empty; does **not** touch stock yet — that's the `/stock` endpoint wired up in v2 |
| GET | `/api/orders/history/{userId}` | list PLACED orders for a user | |

### 5.3 The one inter-service call in v1

`order-service` → `product-service`, via `RestClient` resolved through Eureka (`http://product-service/api/products/{id}`, with Eureka's client-side load balancing resolving the actual host:port). This happens on add-to-cart, to snapshot name/price. This is the pattern every later inter-service call in v2–v4 will reuse, so it's worth getting the error handling right here: if product-service is unreachable or returns 404, order-service should return a clean 400/404 to the frontend, not a raw exception stack trace.

---

## 6. frontend-service

Spring Boot MVC + Thymeleaf, no JS. No entities, no database. Pure controller → RestClient → template flow, same shape as the `book-partner-frontend` module in your reference guide.

### 6.1 Pages / controllers

| Route | Calls | Template |
|---|---|---|
| `GET /` | product-service `GET /api/products` | `home.html` — product grid |
| `GET /products/{id}` | product-service `GET /api/products/{id}` | `product-detail.html` |
| `GET /register`, `POST /register` | user-service `POST /api/users/register` | `register.html` |
| `GET /login`, `POST /login` | user-service `POST /api/users/login` | `login.html` — on success, store token in the frontend's own server-side HTTP session (`HttpSession`), never expose it to the browser as a cookie/localStorage value the user can read |
| `GET /cart` | order-service `GET /api/orders/cart/{userId}` | `cart.html` |
| `POST /cart/add` | order-service `POST /api/orders/cart/{userId}/items` | redirect back to product detail |
| `POST /checkout` | order-service `POST /api/orders/checkout/{userId}` | `order-confirmation.html` |
| `GET /orders` | order-service `GET /api/orders/history/{userId}` | `order-history.html` |

`userId` for all these calls comes from the frontend's `HttpSession`, populated at login by calling `GET /api/users/me` with the stored token.

---

## 7. End-to-end flow (for sanity-checking the whole thing works)

```
User registers → user-service creates row, returns 201
User logs in → user-service returns token → frontend stores in HttpSession
User browses → frontend calls product-service, renders grid
User adds to cart → frontend → order-service → order-service calls product-service
  → order-service creates/updates CART order + OrderItem with snapshot fields
User checks out → frontend → order-service → CART order flips to PLACED
User views order history → frontend → order-service → list of PLACED orders
```

If you can walk through this exact sequence manually (Postman or the actual frontend) end to end without an error, v1 is done.

---

## 8. Known limitations (intentional, fixed in later versions)

- No API Gateway — frontend hits each service directly (v2)
- Sessions are in-memory in user-service, lost on restart, no real JWT (v2)
- No stock decrement at checkout — stock is display-only (v2)
- No inter-service failure handling beyond basic 4xx/5xx passthrough — no retries/circuit breakers (v5)
- No recommendations, no payment, no notifications (v3, v4, v4)

---

## 9. Testing checklist before calling v1 "done"

- [ ] eureka-server dashboard shows all 4 other services registered
- [ ] product-service: create category, create 3+ products, list, get-by-id, update, delete all work via Postman
- [ ] user-service: register succeeds, duplicate email returns 409, login returns a token, bad password returns 401, `/me` resolves correctly with the token
- [ ] order-service: add-to-cart correctly snapshots name/price from a live product-service call, checkout flips status, history returns only PLACED orders
- [ ] frontend: full manual walkthrough of Section 7 works with zero errors in the browser
