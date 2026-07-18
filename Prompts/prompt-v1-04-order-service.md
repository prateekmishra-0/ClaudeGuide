# PROMPT 4 (FINAL) — order-service

> Paste everything in the code block below directly into Amazon Q as a single prompt.
> This service calls product-service over HTTP via a declarative Feign client. product-service must already be built and registered with eureka-server before you test this end to end — but you can still generate, compile, and unit-test this service on its own first (that's the point of the mocked-Feign-client tests below).
> Assumes the project was already generated via Spring Initializr per the settings given earlier (Spring Boot 4.1.0, Group com.ecommerce, Artifact order-service, Package com.ecommerce.orderservice, dependencies: Spring Web, Spring Data JPA, Validation, Eureka Discovery Client, OpenFeign, Spring Cloud LoadBalancer, PostgreSQL Driver, Lombok).
> If Amazon Q's response gets cut off, the natural split point is: (4a) entities + repositories + cart-only endpoints, (4b) the Feign client + checkout logic, (4c) the unit tests as a separate follow-up. Say so and I'll write the multi-part version instead.

```
I have already generated this Spring Boot project via Spring Initializr — do not modify, add, or remove anything in pom.xml, and do not change the Spring Boot or Spring Cloud version:

- Spring Boot 4.1.0, Spring Cloud release train 2025.1.2 (already configured in pom.xml)
- Group: com.ecommerce, Artifact: order-service, Package: com.ecommerce.orderservice
- Dependencies present: Spring Web, Spring Data JPA, Validation, Eureka Discovery Client, OpenFeign, Spring Cloud LoadBalancer, PostgreSQL Driver, Lombok (spring-boot-starter-test is included by default, no action needed for it)
- Configuration format: application.yml (already exists, currently empty/default)

Your task is ONLY to write the application logic, configuration, and tests on top of this existing project. Do not touch pom.xml. Do not suggest adding any dependency beyond what's listed above.

MAIN APPLICATION CLASS:
- File: src/main/java/com/ecommerce/orderservice/OrderServiceApplication.java
- Annotations: @SpringBootApplication, @EnableDiscoveryClient, @EnableFeignClients

CONFIGURATION FILE:
- File: src/main/resources/application.yaml
- server.port: 8083
- spring.application.name: order-service
- spring.datasource.url: jdbc:postgresql://localhost:5432/order_db
- spring.datasource.username: postgres
- spring.datasource.password: root
- spring.jpa.hibernate.ddl-auto: update
- spring.jpa.show-sql: true
- eureka.client.service-url.defaultZone: http://localhost:8761/eureka

ENTITIES (package: com.ecommerce.orderservice.entity):

Entity: Order
- id: Long, @Id, @GeneratedValue(strategy = GenerationType.IDENTITY)
- userId: Long, @NotNull — plain reference, no cross-service foreign key
- status: String, @NotNull, @Column(length = 30) — valid values are the literal strings "CART" and "PLACED" only for this version; validate this in code (a simple check, not a JPA @Enumerated type). Column is deliberately sized at 30 rather than the shorter length these two values need, to leave headroom for longer status values later versions will introduce (e.g. PAYMENT_PENDING_RETRY) without requiring a schema change then.
- createdAt: LocalDateTime — set only inside @PrePersist, @Column(updatable = false)
- updatedAt: LocalDateTime — set inside both @PrePersist and @PreUpdate

Entity: OrderItem
- id: Long, @Id, @GeneratedValue(strategy = GenerationType.IDENTITY)
- order: Order, @ManyToOne(fetch = FetchType.LAZY), @JoinColumn(name = "order_id", nullable = false) — this IS a real JPA relationship, unlike product-service's plain-column pattern
- productId: Long, @NotNull — plain reference to product-service's Product, no JPA relationship (different schema/service)
- productNameSnapshot: String, @Column(length = 100) — copied from product-service at the moment the item is added to cart, not read live afterward
- priceSnapshot: BigDecimal, @NotNull — copied at the same moment as the name snapshot
- quantity: Integer, @NotNull, @Min(1)

Use Lombok @Getter @Setter @NoArgsConstructor @AllArgsConstructor on both entities.

REPOSITORIES (package: com.ecommerce.orderservice.repository):
- OrderRepository extends JpaRepository<Order, Long>
  - Add method: Optional<Order> findByUserIdAndStatus(Long userId, String status)
  - Add method: List<Order> findByUserIdAndStatusNot(Long userId, String excludedStatus) — used for order history (everything that isn't a live CART)
- OrderItemRepository extends JpaRepository<OrderItem, Long>
  - Add method: Optional<OrderItem> findByOrderAndProductId(Order order, Long productId) — used to detect whether the product being added is already a line item in this cart, so a repeat add-to-cart increments quantity instead of creating a duplicate row

DTOs (package: com.ecommerce.orderservice.dto):
- AddItemRequest: productId (Long, @NotNull), quantity (Integer, @NotNull, @Min(1))
- ProductResponse: id (Long), name (String), price (BigDecimal), stockQuantity (Integer) — this is the shape used to deserialize product-service's GET /api/products/{id} response; only include fields we actually need
- OrderItemResponse: id (Long), productId (Long), productNameSnapshot (String), priceSnapshot (BigDecimal), quantity (Integer)
- OrderResponse: id (Long), userId (Long), status (String), items (List<OrderItemResponse>), createdAt (LocalDateTime)

FEIGN CLIENT (package: com.ecommerce.orderservice.client):
- Interface: ProductServiceClient, annotated @FeignClient(name = "product-service")
- Method: @GetMapping("/api/products/{id}") ProductResponse getProduct(@PathVariable("id") Long id)
- This is purely a declarative interface — no implementation body, no manual URL/host construction. Spring Cloud OpenFeign generates the implementation at runtime, resolving "product-service" through Eureka.
- Error handling: Feign throws FeignException subtypes on non-2xx responses. Do NOT write a custom ErrorDecoder for this version — instead, catch feign.FeignException.NotFound specifically at the call site in the service layer (see OrderService below) and translate it into our own ProductNotFoundException. Let any other FeignException subtype propagate as-is for now — no retry or circuit breaker logic in this version.

SERVICE LAYER (package: com.ecommerce.orderservice.service):
- Class: OrderService (a real @Service class here, not controller-calls-repository-directly like product-service — this service has enough logic to warrant it)
- Method: Order getOrCreateCart(Long userId) — finds the existing CART order for the user via findByUserIdAndStatus, or creates and saves a new one with status "CART" if none exists
- Method: OrderItem addItemToCart(Long userId, AddItemRequest request) — calls getOrCreateCart to get the current CART order, then calls orderItemRepository.findByOrderAndProductId(cart, request.getProductId()) to check whether this product already has a line item in the cart.
    - If found: increment its quantity by request.getQuantity() (do NOT call ProductServiceClient again — keep the existing productNameSnapshot/priceSnapshot as they were at the time of the first add), save, return it.
    - If not found: proceed exactly as before — call ProductServiceClient.getProduct(request.getProductId()), catch feign.FeignException.NotFound and re-throw as ProductNotFoundException, create a new OrderItem with snapshot fields from the Feign response, save, return it.
  Adding the same product to the same cart twice must always result in one line item whose quantity is the sum of both adds — never two rows for the same productId in one cart order.
- Method: void removeItemFromCart(Long itemId) — deletes the OrderItem by id; throw OrderItemNotFoundException if it doesn't exist
- Method: Order checkout(Long userId) — finds the CART order via findByUserIdAndStatus; if none exists OR it has zero items, throw EmptyCartException; otherwise set status to "PLACED", save, return it
- Method: List<Order> getOrderHistory(Long userId) — returns findByUserIdAndStatusNot(userId, "CART")

CONTROLLER (package: com.ecommerce.orderservice.controller):

OrderController — base path /api/orders
- GET /api/orders/cart/{userId} — calls getOrCreateCart, maps to OrderResponse, returns 200 (an empty-items cart is a valid 200, not a 404)
- POST /api/orders/cart/{userId}/items — @Valid @RequestBody AddItemRequest, calls addItemToCart, returns 201 with the updated cart as OrderResponse (re-fetch the cart after adding, don't just return the single item)
- DELETE /api/orders/cart/{userId}/items/{itemId} — calls removeItemFromCart, returns 204
- POST /api/orders/checkout/{userId} — calls checkout, returns 200 with OrderResponse
- GET /api/orders/history/{userId} — calls getOrderHistory, returns 200 with List<OrderResponse>

EXCEPTION HANDLING (package: com.ecommerce.orderservice.exception):
- Custom exception: ProductNotFoundException extends RuntimeException (constructor takes a String message) — thrown when the Feign call to product-service returns 404
- Custom exception: OrderItemNotFoundException extends RuntimeException (constructor takes a String message)
- Custom exception: EmptyCartException extends RuntimeException (constructor takes a String message)
- Class: GlobalExceptionHandler, annotated @RestControllerAdvice
  - @ExceptionHandler(MethodArgumentNotValidException.class) → 400, return a body listing each invalid field and its validation message
  - @ExceptionHandler(ProductNotFoundException.class) → 404, return { "message": <exception message> }
  - @ExceptionHandler(OrderItemNotFoundException.class) → 404, return { "message": <exception message> }
  - @ExceptionHandler(EmptyCartException.class) → 400, return { "message": <exception message> }
  - @ExceptionHandler(Exception.class) → 500, generic fallback, return { "message": "An unexpected error occurred" }

UNIT TESTS (package: com.ecommerce.orderservice.service, in src/test/java):

Test class: OrderServiceTest
- Annotated @ExtendWith(MockitoExtension.class) — this is a plain unit test, NOT @SpringBootTest or @WebMvcTest, since we're testing OrderService's own logic directly, not the HTTP layer
- @Mock OrderRepository
- @Mock OrderItemRepository
- @Mock ProductServiceClient — this is the Feign client interface; mocking it directly with Mockito means product-service does not need to be running for any of these tests
- @InjectMocks OrderService
1. addItemToCart_whenProductServiceClientThrowsNotFound_throwsProductNotFoundException — mock productServiceClient.getProduct(anyLong()) to throw feign.FeignException.NotFound (use a minimal valid constructor for that exception type), call addItemToCart, assert that our own ProductNotFoundException is thrown, not the raw FeignException
2. addItemToCart_whenProductExists_createsOrderItemWithCorrectSnapshotFields — mock productServiceClient.getProduct() to return a ProductResponse with a known name and price, mock orderRepository.findByUserIdAndStatus() to return an existing CART order, call addItemToCart, capture the OrderItem passed to orderItemRepository.save() with an ArgumentCaptor, assert its productNameSnapshot and priceSnapshot match exactly what the mocked ProductResponse returned
3. checkout_whenCartHasZeroItems_throwsEmptyCartException — mock findByUserIdAndStatus to return a CART order with an empty items list, call checkout, assert EmptyCartException is thrown, and assert orderRepository.save() was NEVER called (Mockito.verify(...,never()))
4. checkout_whenCartHasItems_setsStatusToPlacedAndSaves — mock findByUserIdAndStatus to return a CART order with at least one item, call checkout, capture the Order passed to save(), assert its status field equals "PLACED"
5. getOrCreateCart_whenNoCartExists_createsAndSavesNewCartOrder — mock findByUserIdAndStatus to return Optional.empty(), call getOrCreateCart, assert orderRepository.save() was called exactly once with a new Order whose status is "CART" and userId matches the input
6. addItemToCart_whenProductAlreadyInCart_incrementsExistingQuantityInsteadOfCreatingDuplicate — mock findByUserIdAndStatus to return an existing CART order, mock findByOrderAndProductId to return an existing OrderItem with quantity 2, call addItemToCart with quantity 3, capture the OrderItem passed to save(), assert quantity == 5, and verify productServiceClient.getProduct() was NEVER called (Mockito.verify(..., never()))

CONSTRAINTS:
- Do not decrement or otherwise touch product-service's stock in this version — checkout only flips status, nothing more. Stock mutation is a later version's addition.
- Do not add a custom ErrorDecoder, retry, circuit breaker, or timeout configuration on the Feign client — a plain, unguarded call with the one specific NotFound catch described above is correct for this version.
- Do not add authentication/authorization checks anywhere — userId is taken directly from the path variable with no verification in this version.
- Do not add any endpoints beyond the five listed above.
- Do not start a Spring application context anywhere in OrderServiceTest — plain Mockito only, so these tests pass identically whether product-service is running or not.
- Output every file in full: application.yml, both entities, both repositories, all four DTOs, the ProductServiceClient Feign interface, OrderService, OrderController, all three custom exceptions, GlobalExceptionHandler, and OrderServiceTest. Do NOT output pom.xml — it already exists and must not change.
- Do not add any status values beyond "CART" and "PLACED" in this version, even though the column is sized for longer future values — the extra length is schema headroom only, not a signal to add new statuses now.

VERIFICATION (tell me how to confirm this works):
- List the exact curl or Postman requests to test, IN THIS ORDER since product-service must already have data: create a category and 2 products in product-service first, then add both to a user's cart via order-service, view the cart and confirm the name/price snapshots match product-service's data at that moment, remove one item, checkout, confirm status is PLACED, view order history, then attempt to check out an empty cart for a different userId and confirm EmptyCartException triggers (400), then attempt to add a non-existent productId to a cart and confirm ProductNotFoundException triggers (404).
- Also tell me the exact Maven command to run just OrderServiceTest, confirm it passes with product-service NOT running (to prove the mocked isolation actually works), and describe what a fully-passing console output should look like.
```