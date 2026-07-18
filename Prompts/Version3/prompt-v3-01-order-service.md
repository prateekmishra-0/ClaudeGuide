This is NOT a new project — you're continuing in the existing order-service project from v1/v2. Do not run Spring Initializr again. No dependency changes are needed for this version — do not modify pom.xml at all, do not suggest any addition.

Existing files in this project (do not regenerate these from scratch — you are extending/modifying them):
- Order, OrderItem entities — unchanged, do not touch
- OrderRepository, OrderItemRepository — EXTENDED below (one new repository method), everything else unchanged
- AddItemRequest, OrderItemResponse, OrderResponse, ProductResponse, StockAdjustRequest DTOs — unchanged, do not touch
- ProductServiceClient Feign interface — unchanged, do not touch
- OrderService — unchanged, do not touch, no existing method's logic changes in this version
- OrderController — EXTENDED below (one new endpoint), no existing method changes
- All existing exceptions (ProductNotFoundException, OrderItemNotFoundException, EmptyCartException, InsufficientStockException) — unchanged, do not touch
- GlobalExceptionHandler — unchanged, do not touch, no new exception type is introduced in this version
- InternalSecretFilter, FilterConfig, FeignInternalSecretInterceptor (from v2) — unchanged, do not touch
- OrderServiceTest — unchanged, do not touch (see note on tests below)

No configuration file change in this version — application.yaml stays exactly as it is from v2.

MODIFY: OrderItemRepository (package: com.ecommerce.orderservice.repository)
- Add method: List<OrderItem> findByProductIdAndOrder_Status(Long productId, String status)
  — this queries across ALL orders' items, filtered by productId and the parent Order's status; Spring Data derives this via the nested property path Order_Status through the existing @ManyToOne relationship, no custom @Query needed
- Do not add any other repository method. Do not touch OrderRepository.

MODIFY: OrderService (package: com.ecommerce.orderservice.service)
- Add new method: List<Order> getOrdersContainingProduct(Long productId)
    1. Call orderItemRepository.findByProductIdAndOrder_Status(productId, "PLACED")
    2. From the resulting OrderItem list, extract the distinct parent Order objects (each OrderItem's .getOrder()) — dedupe in case an order somehow has multiple line items for the same product, though that shouldn't happen given v1's cart-merge logic
    3. Return the distinct list of Orders
- Do not modify any existing method (getOrCreateCart, addItemToCart, removeItemFromCart, checkout, getOrderHistory).

MODIFY: OrderController (package: com.ecommerce.orderservice.controller)
- Add new endpoint: GET /api/orders/containing-product/{productId}
    - Calls orderService.getOrdersContainingProduct(productId)
    - Maps each Order to the existing OrderResponse DTO (same mapping pattern already used by every other endpoint in this controller)
    - Returns 200 with List<OrderResponse> — an empty list is a valid 200, not a 404, since "no orders contain this product yet" is a normal, expected state for a new product
- Do not modify any existing endpoint method in this controller.

UNIT TEST ADDITION (package: com.ecommerce.orderservice.service, in src/test/java):

Add to the existing OrderServiceTest class (do not create a new test class, do not modify any of the existing 8 tests):
9. getOrdersContainingProduct_whenOrdersExist_returnsDistinctOrdersContainingThatProduct — mock orderItemRepository.findByProductIdAndOrder_Status() to return a list of OrderItems whose .getOrder() values include a duplicate (two items pointing to the same Order instance, plus one item pointing to a different Order), call getOrdersContainingProduct, assert the returned list has exactly 2 distinct orders (not 3)
10. getOrdersContainingProduct_whenNoOrdersContainProduct_returnsEmptyList — mock the repository call to return an empty list, call getOrdersContainingProduct, assert the returned list is empty (not null, not an exception)

CONSTRAINTS:
- Do not add authentication/authorization (userId ownership) checks anywhere in this endpoint — same stated known limitation as the rest of order-service currently has, not addressed here.
- Do not add pagination to this new endpoint — a plain List<OrderResponse> is correct for this version's scale.
- Do not touch InternalSecretFilter, FilterConfig, or FeignInternalSecretInterceptor — this is a purely additive read endpoint, the existing internal-secret mechanism from v2 already covers it automatically since the filter has no whitelist.
- Do not modify Order or OrderItem entities — this is a new query method on existing data, not a schema change.
- Output only the files that actually change: the modified OrderItemRepository (full file), the modified OrderService (full file), the modified OrderController (full file), and the modified OrderServiceTest (full file). Do not regenerate or restate any entity, DTO, the Feign client, any exception, GlobalExceptionHandler, InternalSecretFilter, FilterConfig, or FeignInternalSecretInterceptor.

VERIFICATION (tell me how to confirm this works):
- With order-service running standalone (product-service does NOT need to be running for this endpoint specifically, since it doesn't call out anywhere): place a few test orders through the full checkout flow first (via Postman, hitting cart-add then checkout for 2-3 different products across 2-3 different users, so you have real PLACED orders with known productIds in them).
- Send GET /api/orders/containing-product/{productId} directly to port 8083 (with the correct X-Internal-Secret header, since the v2 filter still applies) for a productId you know is in at least two placed orders — confirm it returns exactly those orders, no duplicates, no CART-status orders included.
- Send the same request for a productId that exists in product-service but has never been ordered — confirm it returns an empty list with 200, not a 404 or 500.
- Send the same request with no X-Internal-Secret header — confirm 401, proving the v2 filter still guards this new endpoint with no special-casing needed.
- Also tell me the exact Maven command to run just OrderServiceTest, confirm all 10 tests pass, and describe what a fully-passing console output should look like.