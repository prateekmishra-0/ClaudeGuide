I have already generated this Spring Boot project via Spring Initializr — do not modify, add, or remove anything in pom.xml, and do not change the Spring Boot or Spring Cloud version:

- Spring Boot 4.1.0, Spring Cloud release train 2025.1.2 (already configured in pom.xml)
- Group: com.ecommerce, Artifact: payment-service, Package: com.ecommerce.paymentservice
- Dependencies present: Spring Web, Spring Data JPA, PostgreSQL Driver, Eureka Discovery Client, Lombok (spring-boot-starter-test is included by default, no action needed for it)
- Configuration format: application.yml (already exists, currently empty/default)
- This service HAS a database — Postgres, database name payment_db. It does NOT call any other service — it has no Feign client, no @EnableFeignClients needed.

Your task is ONLY to write the application logic and configuration on top of this existing project. Do not touch pom.xml. Do not suggest adding any dependency beyond what's listed above.

MAIN APPLICATION CLASS:
- File: src/main/java/com/ecommerce/paymentservice/PaymentServiceApplication.java
- Annotations: @SpringBootApplication, @EnableDiscoveryClient
- Do NOT add @EnableFeignClients — this service never calls another service.

CONFIGURATION FILE:
- File: src/main/resources/application.yaml
- server.port: 8085
- spring.application.name: payment-service
- spring.datasource.url: jdbc:postgresql://localhost:5432/payment_db (username/password: match whatever local Postgres credentials the other services already use in their own application.yml files — same values, don't invent new ones)
- spring.jpa.hibernate.ddl-auto: update
- spring.jpa.show-sql: true
- eureka.client.service-url.defaultZone: http://localhost:8761/eureka
- internal.secret: arhjYkqjrB6INMZvZjOApL6jk0b6LWJVnsEYl29kMY0= — this MUST be the exact same value already present in api-gateway's, product-service's, user-service's, and order-service's application.yml files. Do not generate a new value.

ENTITY (package: com.ecommerce.paymentservice.entity):
- Class: Payment
    - id: Long, PK, auto-generated (IDENTITY)
    - orderId: Long, @NotNull
    - amount: BigDecimal, @NotNull, @DecimalMin("0.01")
    - status: String, @NotBlank, @Column(length = 20) — value will be either "SUCCESS" or "FAILED", set in code, not a Java enum type, plain validated String is fine at this scale
    - createdAt: LocalDateTime, set once in a @PrePersist method, not client-settable
      Use Lombok @Getter @Setter @NoArgsConstructor @AllArgsConstructor on this entity, same as every other entity in this project.

REPOSITORY (package: com.ecommerce.paymentservice.repository):
- Interface: PaymentRepository extends JpaRepository<Payment, Long>
- One custom method: Optional<Payment> findByOrderId(Long orderId)
- Do not add any other query method.

DTOs (package: com.ecommerce.paymentservice.dto):
- PaymentRequest: orderId (Long, @NotNull), amount (BigDecimal, @NotNull, @DecimalMin("0.01")) — request body for POST /api/payments
- PaymentResponse: id (Long), orderId (Long), amount (BigDecimal), status (String), createdAt (LocalDateTime) — used as the response for BOTH endpoints below. The architecture note describing POST's response as just {"status": ..., "paymentId": ...} is a subset of this DTO's fields — returning the fuller shape is a deliberate, harmless choice, not scope creep, since order-service's own client-side DTO only needs to declare the fields it actually reads and Jackson will ignore the rest.

EXCEPTION (package: com.ecommerce.paymentservice.exception):
- Class: PaymentNotFoundException extends RuntimeException — thrown when GET /api/payments/order/{orderId} finds no matching payment

HANDLER (package: com.ecommerce.paymentservice.exception):
- Class: GlobalExceptionHandler, @RestControllerAdvice
    - MethodArgumentNotValidException → 400, field errors in the response body
    - PaymentNotFoundException → 404, return { "message": <exception message> }
    - generic Exception → 500, return { "message": "An unexpected error occurred" }

SECURITY — INCOMING (package: com.ecommerce.paymentservice.security):
- Class: InternalSecretFilter, extends OncePerRequestFilter, @Component
- Reads internal.secret via @Value
- In doFilterInternal: read header "X-Internal-Secret"; if missing or doesn't exactly match the configured value, write HTTP 401 directly, content type application/json, body {"message": "Forbidden — direct access not permitted"}, return without calling filterChain.doFilter(); otherwise call filterChain.doFilter()
- Applies to EVERY endpoint in this service, no whitelist — including POST /api/payments

NEW CLASS (package: com.ecommerce.paymentservice.config):
- Class: FilterConfig, @Configuration
- A @Bean method of type FilterRegistrationBean<InternalSecretFilter>, .setOrder(Ordered.HIGHEST_PRECEDENCE)

SERVICE LAYER (package: com.ecommerce.paymentservice.service):
- Class: PaymentService, @Service

Method: PaymentResponse processPayment(PaymentRequest request)
1. Determine status: "FAILED" if request.getAmount() <= 0 OR if the amount's cents value ends in .13 (e.g. 100.13, 49.13 — compare the last two digits of the unscaled value; document this rule plainly in a code comment so it doesn't look like a mystery bug), otherwise "SUCCESS"
2. Persist a new Payment with orderId, amount, the determined status, and createdAt set via @PrePersist — persist it EITHER way, success or failure
3. Return the saved entity mapped to PaymentResponse

Method: PaymentResponse getPaymentByOrderId(Long orderId)
1. Call paymentRepository.findByOrderId(orderId)
2. If empty, throw PaymentNotFoundException
3. Map the found entity to PaymentResponse and return it

CONTROLLER (package: com.ecommerce.paymentservice.controller):
- Class: PaymentController, base path /api/payments
- POST /api/payments — body: PaymentRequest, @Valid — calls processPayment, returns 201 with PaymentResponse
- GET /api/payments/order/{orderId} — calls getPaymentByOrderId, returns 200 with PaymentResponse (404 via PaymentNotFoundException if none exists)

CONSTRAINTS:
- Do not add retry, circuit breaker, or timeout logic anywhere — that's v5.
- Do not add caching anywhere.
- Do not add a Feign client of any kind — this service never calls another service in v4.
- Do not use Math.random() anywhere — the FAILED conditions must be fully deterministic (amount <= 0, or cents ending in .13), so tests are reproducible.
- Do not add authentication/authorization (role checks) — that's v5.
- Do not add unit tests in this prompt — manual verification via the steps below is sufficient for this version, same discipline recommendation-service's v3 prompt used.
- Output every file in full: application.yml, Payment entity, PaymentRepository, both DTOs, PaymentNotFoundException, GlobalExceptionHandler, InternalSecretFilter, FilterConfig, PaymentService, PaymentController, and PaymentServiceApplication.java. Do NOT output pom.xml.

VERIFICATION (tell me how to confirm this works):
- Confirm payment-service registers with Eureka on startup (check the eureka-server dashboard).
- Send POST /api/payments directly to port 8085 (with the correct X-Internal-Secret header) with a normal amount like {"orderId": 1, "amount": 49.99} — confirm status SUCCESS, a Payment row is persisted, response includes an id.
- Send the same request with an amount ending in .13, e.g. {"orderId": 2, "amount": 49.13} — confirm status FAILED, still persisted.
- Send the same request with amount 0 or negative — confirm status FAILED.
- Send GET /api/payments/order/1 — confirm it returns the payment created above with the correct fields.
- Send GET /api/payments/order/999 (an orderId that was never paid) — confirm 404, not an empty 200.
- Send any of the above with NO X-Internal-Secret header — confirm 401, request never reaches the controller.