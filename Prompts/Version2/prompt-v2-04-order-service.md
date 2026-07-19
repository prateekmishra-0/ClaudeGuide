This is NOT a new project — you're continuing in the existing order-service project from v1. Do not run Spring Initializr again. No dependency changes are needed for this version — do not modify pom.xml at all, do not suggest any addition.

Existing files in this project (do not regenerate these from scratch — you are extending/modifying them):
- Order, OrderItem entities — unchanged, do not touch
- OrderRepository, OrderItemRepository — unchanged, do not touch
- AddItemRequest, OrderItemResponse, OrderResponse DTOs — unchanged, do not touch
- ProductServiceClient Feign interface — EXTENDED below, not replaced
- ProductResponse DTO — unchanged, do not touch
- OrderController — unchanged, do not touch, no endpoint signatures change in this version
- OrderItemNotFoundException, EmptyCartException — unchanged, do not touch
- ProductNotFoundException — unchanged, do not touch

CONFIGURATION FILE CHANGE:
- File: src/main/resources/application.yaml
- Add this one new key (keep every existing key exactly as it is):
  internal.secret: arhjYkqjrB6INMZvZjOApL6jk0b6LWJVnsEYl29kMY0= — this exact string must be the SAME value already placed in api-gateway's, user-service's, and product-service's config. Do not generate a new value.

PART A — INCOMING: internal-secret checker filter

NEW CLASS (package: com.ecommerce.orderservice.security):
- Class: InternalSecretFilter, extends OncePerRequestFilter, annotated @Component
- Reads internal.secret from application.yaml via @Value
- In doFilterInternal(request, response, filterChain):
  1. Read header "X-Internal-Secret" from the request
  2. If missing OR does not exactly equal the configured internal.secret value, write HTTP status 401 directly to the response, set content type application/json, write body {"message": "Forbidden — direct access not permitted"}, and RETURN without calling filterChain.doFilter()
  3. If it matches, call filterChain.doFilter(request, response)
- Applies to EVERY endpoint in this service with NO exceptions, no whitelist.
- Unrelated to JWTs — order-service never sees a JWT, it trusts X-User-Id/X-User-Email headers forwarded by the gateway, and this filter is what makes that trust reasonable (it guarantees the request came through the gateway in the first place).

NEW CLASS (package: com.ecommerce.orderservice.config):
- Class: FilterConfig, annotated @Configuration
- A @Bean method of type FilterRegistrationBean<InternalSecretFilter>, registering it with .setOrder(Ordered.HIGHEST_PRECEDENCE)

PART B — OUTGOING: attach the secret on order-service's own calls to product-service

NEW CLASS (package: com.ecommerce.orderservice.config):
- Class: FeignInternalSecretInterceptor, implements feign.RequestInterceptor, annotated @Component
- Reads internal.secret from application.yaml via @Value
- Override apply(RequestTemplate template): unconditionally add header "X-Internal-Secret" with the configured value to every outgoing Feign request template
- This applies automatically to ProductServiceClient's calls (getProduct and the new adjustStock method below) since Spring Cloud OpenFeign picks up any RequestInterceptor bean globally — no per-client wiring needed
- This is unrelated to the JWT and unrelated to X-User-Id — it exists purely so product-service's own InternalSecretFilter (built in a separate prompt) accepts these calls; without this interceptor, every Feign call from order-service to product-service would be rejected with 401 the moment product-service's filter is active

PART C — checkout flow: stock validation (unchanged from prior scope, unaffected by parts A/B)

NEW DTO (package: com.ecommerce.orderservice.dto):
- StockAdjustRequest: delta (Integer)

MODIFY: ProductServiceClient (package: com.ecommerce.orderservice.client)
- Add: @PutMapping("/api/products/{id}/stock") ProductResponse adjustStock(@PathVariable("id") Long id, @RequestBody StockAdjustRequest request)
- No ErrorDecoder; error translation happens at the call site in OrderService.

NEW EXCEPTION (package: com.ecommerce.orderservice.exception):
- InsufficientStockException extends RuntimeException (constructor takes a String message)

MODIFY: GlobalExceptionHandler (package: com.ecommerce.orderservice.exception)
- Add: @ExceptionHandler(InsufficientStockException.class) → 409, return { "message": <exception message> }
- Do not modify any existing handler.

MODIFY: OrderService (package: com.ecommerce.orderservice.service)
- Method checkout(Long userId) — replace the entire v1 method body:
  1. Find the CART order via findByUserIdAndStatus(userId, "CART"); if none exists OR zero items, throw EmptyCartException
  2. VALIDATION PASS — for EVERY item, call productServiceClient.getProduct(item.getProductId()), compare returned stockQuantity against requested quantity. If any item's quantity exceeds available stock, throw InsufficientStockException naming the product and available amount — before mutating anything
  3. MUTATION PASS — only after every item passes validation, for EVERY item call productServiceClient.adjustStock(item.getProductId(), new StockAdjustRequest(-item.getQuantity())). Wrap each call in try/catch for feign.FeignException.Conflict (race-condition case); if caught, re-throw as InsufficientStockException noting the race
  4. Set status to "PLACED", save, return it
- Do not change getOrCreateCart, addItemToCart, removeItemFromCart, or getOrderHistory.

UPDATE EXISTING TEST: OrderServiceTest (package: com.ecommerce.orderservice.service, in src/test/java)
- Keep tests 1, 2, 3, 5 exactly as they are
- Test 4 needs updating: mock productServiceClient.getProduct() with sufficient stock and productServiceClient.adjustStock() to not throw, for each mocked cart item; assertion stays the same
- Add tests 6, 7, 8:
  6. checkout_whenAnyItemHasInsufficientStock_throwsInsufficientStockException_andNeverCallsAdjustStock — two items, item 2 has insufficient stock; assert exception thrown and adjustStock never called for either item
  7. checkout_whenAllItemsHaveSufficientStock_callsAdjustStockForEveryItemWithCorrectNegativeDelta — two items, both sufficient; verify adjustStock called once per item with correct negative delta via ArgumentCaptor
  8. checkout_whenAdjustStockThrowsConflictDuringMutationPass_throwsInsufficientStockException — one item passes validation, adjustStock throws feign.FeignException.Conflict; assert our own InsufficientStockException thrown
- Do NOT add any test for InternalSecretFilter or FeignInternalSecretInterceptor in this class — both are outside the scope of a plain Mockito unit test on OrderService; manual verification (below) covers them.

CONSTRAINTS:
- Do not add retry or circuit breaker logic anywhere — deferred to v5.
- Do not change the order of item processing beyond "all validation before any mutation."
- Do not add authentication/authorization (userId ownership) checks anywhere in this service — a separate, stated known limitation, not addressed here.
- Do not modify OrderController, any entity, any repository, or any DTO other than StockAdjustRequest.
- Do not make InternalSecretFilter and FeignInternalSecretInterceptor share any code or configuration beyond both reading the same internal.secret value independently — one checks incoming requests, the other decorates outgoing ones, they are not related classes.
- Output only the files that actually change: application.yaml (full file), InternalSecretFilter (full file), FilterConfig (full file), FeignInternalSecretInterceptor (full file), StockAdjustRequest (full file), the modified ProductServiceClient (full file), InsufficientStockException (full file), the modified GlobalExceptionHandler (full file), the modified OrderService (full file), and the modified OrderServiceTest (full file). Do not regenerate or restate anything listed as unchanged above.

VERIFICATION (tell me how to confirm this works):
- Filter check: with order-service running standalone, send any request directly to port 8083 with no X-Internal-Secret header — confirm 401. Repeat with the correct header — confirm it proceeds (may still fail for unrelated reasons like a missing cart, but should not be rejected at the filter level).
- Outbound-attach check: with product-service's own InternalSecretFilter already active (built in a separate prompt) and both services running with the SAME internal.secret value: create a product with stockQuantity 5, add 3 to a cart, checkout — confirm success. This proves order-service's FeignInternalSecretInterceptor correctly attached the header on its call to product-service; if it hadn't, this call would fail with a Feign 401 instead of succeeding.
- Stock logic check: confirm stock is now 2 in product-service. Add 3 more of the same product to a new cart, attempt checkout — confirm 409, and confirm stock is still 2 (untouched). Add a 2-item cart where item 1 has enough stock and item 2 does not — checkout, confirm 409, and confirm item 1's stock was NOT decremented either.
- Also tell me the exact Maven command to run just OrderServiceTest, confirm all 8 tests pass, and describe what a fully-passing console output should look like.