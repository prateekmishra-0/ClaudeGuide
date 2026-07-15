# PROMPT 4 — order-service

> Paste everything in the code block below directly into Amazon Q as a single prompt.
> This service calls product-service over HTTP via a declarative Feign client. product-service must already be built and registered with eureka-server before you test this end to end — but you can still generate and compile this service on its own first.
> Assumes the project was already generated via Spring Initializr per the settings above.
> If Amazon Q's response gets cut off, the natural split point is: (4a) entities + repositories + cart-only endpoints (no product-service call), then (4b) the Feign client + checkout logic. Say so and I'll write the two-part version instead.

```
I have already generated this Spring Boot project via Spring Initializr with the following settings — do not modify, add, or remove anything in pom.xml, and do not change the Spring Boot or Spring Cloud version:

- Spring Boot 4.1.0, Spring Cloud release train 2025.1.2 (already configured in pom.xml)
- Group: com.ecommerce, Artifact: order-service, Package: com.ecommerce.orderservice
- Dependencies present: Spring Web, Spring Data JPA, Validation, Eureka Discovery Client, OpenFeign, Spring Cloud LoadBalancer, PostgreSQL Driver, Lombok
- Configuration format: application.yml (already exists, currently empty/default)

Your task is ONLY to write the application logic and configuration on top of this existing project. Do not touch pom.xml. Do not suggest adding any dependency beyond what's listed above.

MAIN APPLICATION CLASS:
- File: src/main/java/com/ecommerce/orderservice/OrderServiceApplication.java
- Annotations: @SpringBootApplication, @EnableDiscoveryClient, @EnableFeignClients

CONFIGURATION FILE:
- File: src/main/resources/application.yml
- server.port: 8083
- spring.application.name: order-service
- spring.datasource.url: jdbc:postgresql://localhost:5432/project_db?currentSchema=order_schema
- spring.datasource.username: postgres
- spring.datasource.password: root
- spring.jpa.hibernate.ddl-auto: update
- spring.jpa.properties.hibernate.default_schema: order_schema
- spring.jpa.show-sql: true
- eureka.client.service-url.defaultZone: http://localhost:8761/eureka

ENTITIES (package: com.ecommerce.orderservice.entity):

Entity: Order
- id: Long, @Id, @GeneratedValue(strategy = GenerationType.IDENTITY)
- userId: Long, @NotNull — plain reference, no cross-service foreign key
- status: String, @NotNull, @Column(length = 20) — valid values are the literal strings "CART" and "PLACED" only; validate this in code (a simple check, not a JPA @Enumerated type)
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
- Method: OrderItem addItemToCart(Long userId, AddItemRequest request) — calls getOrCreateCart, then calls ProductServiceClient.getProduct(request.getProductId()); wrap this call in a try/catch for feign.FeignException.NotFound and re-throw as our own ProductNotFoundException with a clear message; on success, creates an OrderItem with the snapshot fields populated from the Feign response, saves it, returns it
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

CONSTRAINTS:
- Do not decrement or otherwise touch product-service's stock in this version — checkout only flips status, nothing more. Stock mutation is a later version's addition.
- Do not add a custom ErrorDecoder, retry, circuit breaker, or timeout configuration on the Feign client — a plain, unguarded call with the one specific NotFound catch described above is correct for this version.
- Do not add authentication/authorization checks anywhere — userId is taken directly from the path variable with no verification in this version.
- Do not add any endpoints beyond the five listed above.
- Output every file in full: application.yml, both entities, both repositories, all four DTOs, the ProductServiceClient Feign interface, OrderService, OrderController, all three custom exceptions, and GlobalExceptionHandler. Do NOT output pom.xml — it already exists and must not change.

VERIFICATION (tell me how to confirm this works):
- List the exact curl or Postman requests to test, IN THIS ORDER since product-service must already have data: create a category and 2 products in product-service first, then add both to a user's cart via order-service, view the cart and confirm the name/price snapshots match product-service's data at that moment, remove one item, checkout, confirm status is PLACED, view order history, then attempt to check out an empty cart for a different userId and confirm EmptyCartException triggers (400), then attempt to add a non-existent productId to a cart and confirm ProductNotFoundException triggers (404).
```