# SERVICES INDEX

> Permanent, always-present file. Never deleted, never replaced — unlike `architecture-vX.md`, which gets swapped out version by version. Paste this alongside `architecture-vX.md` in every construction or fix prompt, regardless of which version you're on. Keep this under ~50 lines total — if it grows past that, compress descriptions further, don't split it into multiple files (that would defeat its purpose).
>
> Update the **Status** column yourself as each service passes its version's testing checklist. Everything below is pre-filled from the five architecture files already written, so the index is complete even for services not yet built — this lets a service being built now safely reference the *contract* of a service that comes later in the numbering but was already designed.

| Service | Version | Port | Status | Responsibility | Key endpoints |
|---|---|---|---|---|---|
| eureka-server | v1 | 8761 | Planned | service registry | — |
| product-service | v1 | 8081 | Planned | catalog CRUD, category CRUD, stock adjustment | `GET/POST /api/categories`, `GET/POST/PUT/DELETE /api/products`, `PATCH /api/products/{id}/stock` |
| user-service | v1 | 8082 | Planned | registration, login, in-memory session | `POST /api/users/register`, `POST /api/users/login`, `GET /api/users/me` |
| order-service | v1 | 8083 | Planned | cart + checkout (PLACED only, no payment yet) | `GET/POST/DELETE /api/orders/cart/{userId}...`, `POST /api/orders/checkout/{userId}`, `GET /api/orders/history/{userId}` |
| frontend-service | v1 | 8080 | Planned | Thymeleaf UI, no JS | pages only, no API of its own |
| api-gateway | v2 | 8000 | Planned | single entry point, JWT validation | routes `/api/products/**`, `/api/users/**`, `/api/orders/**` |
| recommendation-service | v3 | 8084 | Planned | rule-based → weighted product recommendations | `GET /api/recommendations/product/{id}`, `GET /api/recommendations/user/{id}` |
| payment-service | v4 | 8085 | Planned | mock payment processing | `POST /api/payments`, `GET /api/payments/order/{orderId}` |
| notification-service | v4 | 8086 | Planned | logs order outcome messages | `POST /api/notifications` |

**New in v2 (no new service, existing services change):** order-service adds `GET /api/orders/containing-product/{productId}` in v3, not v2 — listed on order-service's row implicitly; if you want that endpoint visible here too, add it to order-service's endpoint list once v3 is built.

**Status values to use:** `Planned` → `Built, Untested` → `Built & Tested`. Only flip a service to `Built & Tested` once its version's testing checklist fully passes — don't mark it tested just because it compiles.
