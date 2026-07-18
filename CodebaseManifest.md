# CODEBASE MANIFEST

> Permanent, always-present file — paste alongside `promptrule.md`, `services-index.md`, and the current `architecture-vX.md` in every new chat. This file exists purely to record exact package names and class names already committed to code, so a fresh chat can write "ClassX — unchanged, do not touch" correctly without needing the actual prompt files that generated it.
>
> Update this file once per version, right after that version's testing checklist passes — same discipline as updating `services-index.md`'s Status column. Do not paste old `prompt-vN-*.md` files into new chats once this file covers that version; this file replaces them for naming purposes. Entities/fields/endpoints are NOT repeated here — that's `architecture-vX.md`'s job. This file is class names and package names only.

---

## eureka-server — `com.ecommerce.eurekaserver` (v1, Built & Tested)
main: `EurekaServerApplication`
No entities, repos, DTOs, controllers, or security classes — registry only.

---

## product-service — `com.ecommerce.productservice` (v1–v2, Built & Tested)
entity: `Category`, `Product`
repo: `CategoryRepository`, `ProductRepository`
controller: `CategoryController`, `ProductController`
exception: `ProductNotFoundException`, `CategoryNotFoundException`, `InsufficientStockException`, `CategoryAlreadyExistsException`
handler: `GlobalExceptionHandler`
security (v2): `InternalSecretFilter`, `FilterConfig` — both in `com.ecommerce.productservice.security` / `.config`
main: `ProductServiceApplication`

---

## user-service — `com.ecommerce.userservice` (v1–v2, Built & Tested)
entity: `User`
repo: `UserRepository`
dto: `RegisterRequest`, `LoginRequest`, `LoginResponse`, `UserResponse`
config: `BCryptPasswordEncoder` config bean (unnamed class, package `com.ecommerce.userservice.config`)
controller: `UserController`
exception: `EmailAlreadyExistsException`, `InvalidCredentialsException`, `UnauthorizedException`
handler: `GlobalExceptionHandler`
security (v2): `JwtService` (`com.ecommerce.userservice.security`), `InternalSecretFilter` (`.security`), `FilterConfig` (`.config`)
deleted in v2: `SessionStore` (`com.ecommerce.userservice.session`) — no longer exists, do not reference
main: `UserServiceApplication`

---

## order-service — `com.ecommerce.orderservice` (v1–v3, Built & Tested through v2; v3 addition below)
entity: `Order`, `OrderItem`
repo: `OrderRepository`, `OrderItemRepository`
dto: `AddItemRequest`, `ProductResponse`, `OrderItemResponse`, `OrderResponse`, `StockAdjustRequest` (v2)
client: `ProductServiceClient` (`com.ecommerce.orderservice.client`)
service: `OrderService`
controller: `OrderController`
exception: `ProductNotFoundException`, `OrderItemNotFoundException`, `EmptyCartException`, `InsufficientStockException` (v2)
handler: `GlobalExceptionHandler`
security (v2): `InternalSecretFilter` (`.security`), `FilterConfig` (`.config`), `FeignInternalSecretInterceptor` (`.config`)
test: `OrderServiceTest` (`com.ecommerce.orderservice.service`, in `src/test/java`) — 10 tests as of v3
main: `OrderServiceApplication`

**v3 addition:** no new classes — `OrderItemRepository` gained method `findByProductIdAndOrder_Status`, `OrderService` gained method `getOrdersContainingProduct`, `OrderController` gained endpoint `GET /api/orders/containing-product/{productId}`. All existing class names unchanged.

---

## frontend-service — `com.ecommerce.frontendservice` (v1–v3, Built & Tested through v2; v3 addition below)
dto: `ProductResponse`, `ProductPageResponse` (Part A); `RegisterRequest`, `LoginRequest`, `LoginResponse`, `UserResponse`, `AddItemRequest`, `OrderItemResponse`, `OrderResponse` (Part B)
client: `ProductServiceClient`, `UserServiceClient`, `OrderServiceClient` (all in `com.ecommerce.frontendservice.client`; retargeted from individual service names to `"api-gateway"` in v2)
controller: `HomeController`, `ProductController` (Part A); `AuthController`, `CartController`, `OrderController` (Part B)
exception: `ProductNotFoundException` (Part A); `EmailAlreadyExistsException`, `InvalidRegistrationException`, `InvalidCredentialsException`, `EmptyCartException` (Part B); `InsufficientStockException` (v2)
handler: `WebExceptionHandler` (`@ControllerAdvice`, not `@RestControllerAdvice`)
config: `LoginRequiredInterceptor`, its `WebMvcConfigurer` registration, `GlobalModelAttributes` (all Part B, `com.ecommerce.frontendservice.config`)
security (v2): `FeignAuthInterceptor` (`.config`)
templates: `home.html`, `product-detail.html`, `error.html`, `fragments/header.html` (Part A); `register.html`, `login.html`, `cart.html`, `order-confirmation.html`, `orders.html` (Part B)
main: `FrontendServiceApplication`

**v3 addition:** new client `RecommendationServiceClient` (`.client`, targets `"api-gateway"`); `ProductController` and `HomeController` each gained one recommendation-fetching addition (no new class); no new exception (fail-open, caught locally); `product-detail.html` and `home.html` each gained a new section.

---

## api-gateway — `com.ecommerce.apigateway` (v2–v3, Built & Tested through v2; v3 route addition below)
security: `JwtValidationFilter` (`com.ecommerce.apigateway.security`) — includes a nested/private `HttpServletRequestWrapper` subclass for header injection
main: `ApiGatewayApplication`
No entities, repos, DTOs, or controllers — pure routing + filter config in `application.yaml`.

**v3 addition:** no new classes — `application.yaml` gained route `recommendation-route`; `JwtValidationFilter`'s existing whitelist condition gained one new clause (`GET /api/recommendations/product/**`).

---

## recommendation-service — `com.ecommerce.recommendationservice` (v3, Built & Tested — new service)
No entity, no repo, no database — stateless by design.
dto: `ProductResponse`, `OrderItemResponse`, `OrderResponse`, `ProductPageResponse` (all `.dto`)
client: `OrderServiceClient`, `ProductServiceClient` (both `.client`)
service: `RecommendationService`
controller: `RecommendationController`
security: `InternalSecretFilter` (`.security`), `FilterConfig` (`.config`), `FeignInternalSecretInterceptor` (`.config`)
handler: none — service layer never throws upward, no `GlobalExceptionHandler` needed
main: `RecommendationServiceApplication`

---

## payment-service — `com.ecommerce.paymentservice` (v4, Planned — not yet built)
Architecture defines: entity `Payment`; endpoints `POST /api/payments`, `GET /api/payments/order/{orderId}`. No prompt has been written yet, so no committed class names exist below service level. Update this section once v4's payment-service prompt is actually run.

---

## notification-service — `com.ecommerce.notificationservice` (v4, Planned — not yet built)
Architecture defines: no entity, one endpoint `POST /api/notifications`. No prompt has been written yet, so no committed class names exist below service level. Update this section once v4's notification-service prompt is actually run.

---

## Shared config values (do not regenerate, reuse exactly)
- `jwt.secret`: `0bXHZaF1C3tnxFtxM1IcLYCxCQCv1z1a4eX2KKkGlck=` — user-service, api-gateway only
- `internal.secret`: `arhjYkqjrB6INMZvZjOApL6jk0b6LWJVnsEYl29kMY0=` — api-gateway, product-service, user-service, order-service, recommendation-service (payment-service and notification-service will need it too, once built in v4)

## Ports
eureka `8761` · product `8081` · user `8082` · order `8083` · recommendation `8084` · payment `8085` (planned) · notification `8086` (planned) · frontend `8080` · api-gateway `8000`