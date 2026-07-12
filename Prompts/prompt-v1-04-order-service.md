# PROMPT 4 — order-service

> Paste everything in the code block below directly into Amazon Q as a single prompt.
> This service calls product-service over HTTP via Eureka. product-service must already be built and registered with eureka-server before you test this end to end — but you can still generate and compile this service on its own first.
> If Amazon Q's response gets cut off, the natural split point is: (4a) entities + repositories + cart-only endpoints (no product-service call), then (4b) the ProductClient + checkout logic. Say so and I'll write the two-part version instead.

```
Create a Spring Boot microservice from scratch called order-service.

PROJECT SETUP:
- Build tool: Maven
- Java version: 17
- Spring Boot version: 3.2.x
- Group: com.ecommerce
- Artifact: order-service
- Package: com.ecommerce.orderservice
- Packaging: Jar

DEPENDENCIES (pom.xml):
- spring-boot-starter-web
- spring-boot-starter-data-jpa
- spring-boot-starter-validation
- spring-cloud-starter-netflix-eureka-client
- spring-cloud-starter-loadbalancer — required so a RestClient can resolve "product-service" as a logical name through Eureka instead of a hardcoded host:port
- Spring Cloud version: 2023.0.x (compatible with Spring Boot 3.2.x) — add spring-cloud-dependencies BOM in dependencyManagement
- postgresql (runtime scope)
- lombok (optional scope)

MAIN APPLICATION CLASS:
- Class name: OrderServiceApplication
- Location: com.ecommerce.orderservice
- Annotations: @SpringBootApplication, @EnableDiscoveryClient

CONFIGURATION FILE:
- File: src/main/resources/application.yml
- server.port: 8083
- spring.application.name: order-service
- spring.datasource.url: jdbc:postgresql://localhost:5432/ecommerce_db?currentSchema=order_schema
- spring.datasource.username: postgres
- spring.datasource.password: postgres
- spring.jpa.hibernate.ddl-auto: update
- spring.jpa.properties.hibernate.default_schema: order_schema
- spring.jpa.show-sql: true
- eureka.client.service-url.defaultZone: http://localhost:8761/eureka

REST CLIENT BEAN CONFIGURATION (package: com.ecommerce.orderservice.config):
- A @Configuration class exposing a @Bean of type RestClient.Builder, annotated @LoadBalanced, so calls made through it resolve logical service names (e.g. "product-service") via Eureka + spring-cloud-loadbalancer

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

CLIENT SERVICE (package: com.ecommerce.orderservice.client):
- Class: ProductServiceClient — uses the @LoadBalanced RestClient.Builder to build a client targeting base URL "http://product-service"
- Method: ProductResponse getProduct(Long productId) — calls GET /api/products/{id}; if the response is a 404, catch it and throw our own ProductNotFoundException with a clear message; if any other error occurs (connection refused, timeout, 5xx), let it propagate as-is for now — do not add retry or circuit breaker logic, that is a later version's concern

SERVICE LAYER (package: com.ecommerce.orderservice.service):
- Class: OrderService (a real @Service class here, not controller-calls-repository-directly like product-service — this service has enough logic to warrant it)
- Method: Order getOrCreateCart(Long userId) — finds the existing CART order for the user via findByUserIdAndStatus, or creates and saves a new one with status "CART" if none exists
- Method: OrderItem addItemToCart(Long userId, AddItemRequest request) — calls getOrCreateCart, then calls ProductServiceClient.getProduct(request.getProductId()) to get name/price, creates an OrderItem with the snapshot fields populated from that response, saves it, returns it
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
- Custom exception: ProductNotFoundException extends RuntimeException (constructor takes a String message) — thrown by ProductServiceClient
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
- Do not add retry, circuit breaker, or timeout configuration on the ProductServiceClient call — a plain, unguarded call is correct for this version.
- Do not add authentication/authorization checks anywhere — userId is taken directly from the path variable with no verification in this version.
- Do not add any endpoints beyond the five listed above.
- Output every file in full: pom.xml, application.yml, the RestClient config class, both entities, both repositories, all four DTOs, ProductServiceClient, OrderService, OrderController, all three custom exceptions, and GlobalExceptionHandler.

VERIFICATION (tell me how to confirm this works):
- List the exact curl or Postman requests to test, IN THIS ORDER since product-service must already have data: create a category and 2 products in product-service first, then add both to a user's cart via order-service, view the cart and confirm the name/price snapshots match product-service's data at that moment, remove one item, checkout, confirm status is PLACED, view order history, then attempt to check out an empty cart for a different userId and confirm EmptyCartException triggers (400).
```
