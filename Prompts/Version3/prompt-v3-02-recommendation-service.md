I have already generated this Spring Boot project via Spring Initializr — do not modify, add, or remove anything in pom.xml, and do not change the Spring Boot or Spring Cloud version:

- Spring Boot 4.1.0, Spring Cloud release train 2025.1.2 (already configured in pom.xml)
- Group: com.ecommerce, Artifact: recommendation-service, Package: com.ecommerce.recommendationservice
- Dependencies present: Spring Web, Eureka Discovery Client, OpenFeign, Spring Cloud LoadBalancer, Lombok (spring-boot-starter-test is included by default, no action needed for it)
- Configuration format: application.yml (already exists, currently empty/default)
- This service has NO database — no Postgres driver, no Spring Data JPA dependency, and none should be assumed. It is stateless: every call recomputes from live data fetched from order-service and product-service.

Your task is ONLY to write the application logic and configuration on top of this existing project. Do not touch pom.xml. Do not suggest adding any dependency beyond what's listed above.

MAIN APPLICATION CLASS:
- File: src/main/java/com/ecommerce/recommendationservice/RecommendationServiceApplication.java
- Annotations: @SpringBootApplication, @EnableDiscoveryClient, @EnableFeignClients

CONFIGURATION FILE:
- File: src/main/resources/application.yaml
- server.port: 8084
- spring.application.name: recommendation-service
- eureka.client.service-url.defaultZone: http://localhost:8761/eureka
- internal.secret: arhjYkqjrB6INMZvZjOApL6jk0b6LWJVnsEYl29kMY0= — this MUST be the exact same value already present in api-gateway's, user-service's, product-service's, and order-service's application.yml/application.yaml files. Do not generate a new value.

DTOs (package: com.ecommerce.recommendationservice.dto):
- ProductResponse: id (Long), name (String), price (BigDecimal), stockQuantity (Integer), categoryId (Long) — mirrors product-service's Product shape, only the fields actually needed here
- OrderItemResponse: id (Long), productId (Long), productNameSnapshot (String), priceSnapshot (BigDecimal), quantity (Integer)
- OrderResponse: id (Long), userId (Long), status (String), items (List<OrderItemResponse>), createdAt (String) — used to deserialize order-service's responses; createdAt as String since we never parse or compare it here
- ProductPageResponse: content (List<ProductResponse>) — used to deserialize product-service's GET /api/products, which returns a Spring Data Page; only the "content" field is needed, Jackson ignores the rest

FEIGN CLIENTS (package: com.ecommerce.recommendationservice.client):

Interface: OrderServiceClient, annotated @FeignClient(name = "order-service")
- @GetMapping("/api/orders/history/{userId}") List<OrderResponse> getOrderHistory(@PathVariable("userId") Long userId)
- @GetMapping("/api/orders/containing-product/{productId}") List<OrderResponse> getOrdersContainingProduct(@PathVariable("productId") Long productId)
- Purely declarative, no implementation body.

Interface: ProductServiceClient, annotated @FeignClient(name = "product-service")
- @GetMapping("/api/products/{id}") ProductResponse getProductById(@PathVariable("id") Long id)
- @GetMapping("/api/products") ProductPageResponse getProductsByCategory(@RequestParam("categoryId") Long categoryId)
- Purely declarative, no implementation body.

Error handling for BOTH clients: do not write a custom ErrorDecoder. Every call site in RecommendationService (below) must be wrapped so that ANY failure — a Feign exception of any subtype, a timeout, a connection failure — is caught and treated as "no data available," never allowed to propagate as an unhandled exception or a 500. This is the "fail open, fail empty" principle: a broken recommendation is a missing widget, not a broken page.

SECURITY — INCOMING (package: com.ecommerce.recommendationservice.security):
- Class: InternalSecretFilter, extends OncePerRequestFilter, annotated @Component
- Reads internal.secret from application.yaml via @Value
- In doFilterInternal(request, response, filterChain):
    1. Read header "X-Internal-Secret" from the request
    2. If missing OR does not exactly equal the configured internal.secret value, write HTTP status 401 directly to the response, set content type application/json, write body {"message": "Forbidden — direct access not permitted"}, and RETURN without calling filterChain.doFilter()
    3. If it matches, call filterChain.doFilter(request, response)
- Applies to EVERY endpoint in this service with NO exceptions, no whitelist — including GET /api/recommendations/product/{id}, even though api-gateway will treat that one as publicly reachable to end-users; this filter only cares whether the request came through the gateway (which always attaches this header), not whether the end-user needed a JWT.

NEW CLASS (package: com.ecommerce.recommendationservice.config):
- Class: FilterConfig, annotated @Configuration
- A @Bean method of type FilterRegistrationBean<InternalSecretFilter>, registering it with .setOrder(Ordered.HIGHEST_PRECEDENCE)

SECURITY — OUTGOING (package: com.ecommerce.recommendationservice.config):
- Class: FeignInternalSecretInterceptor, implements feign.RequestInterceptor, annotated @Component
- Reads internal.secret from application.yaml via @Value
- Override apply(RequestTemplate template): unconditionally add header "X-Internal-Secret" with the configured value to every outgoing Feign request template
- This applies automatically to all four methods across OrderServiceClient and ProductServiceClient, since Spring Cloud OpenFeign picks up any RequestInterceptor bean globally — no per-client wiring needed
- Without this, every one of this service's four outbound calls would be rejected 401 by order-service's or product-service's own InternalSecretFilter (both already active since v2), and — because of the fail-open-fail-empty rule above — that rejection would look identical to a genuine "no data" case. Worth stating this explicitly in code comments, since it's a real debugging trap.

SERVICE LAYER (package: com.ecommerce.recommendationservice.service):
- Class: RecommendationService, annotated @Service
- Constant: RECOMMENDATION_COUNT = 4 (hardcoded, private static final int)

Method: List<ProductResponse> getRecommendationsForProduct(Long productId)
1. RULE 1 — frequently bought together:
   a. Call orderServiceClient.getOrdersContainingProduct(productId), wrapped in try/catch treating any exception as an empty list
   b. From all returned orders' items, tally every OTHER productId's occurrence count into a Map<Long, Integer> (exclude productId itself)
   c. Sort descending by count, take the top RECOMMENDATION_COUNT productIds — no tie-breaking logic, map iteration order is fine
2. RULE 2 — category fallback, triggered only if step 1 produced fewer than RECOMMENDATION_COUNT results:
   a. Call productServiceClient.getProductById(productId) to resolve its categoryId, wrapped in try/catch — if this call fails, skip Rule 2 entirely and return whatever Rule 1 found (even if fewer than 4)
   b. Call productServiceClient.getProductsByCategory(categoryId), wrapped in try/catch treating failure as an empty list
   c. Exclude the original productId, fill remaining slots (combining with Rule 1's results, no duplicates) up to RECOMMENDATION_COUNT total
3. Resolve each final productId to a full ProductResponse via productServiceClient.getProductById — if any individual resolution fails, skip that one product rather than failing the whole call
4. Return the final list — if everything failed at every step, return an empty list, never throw

Method: List<ProductResponse> getRecommendationsForUser(Long userId)
1. Call orderServiceClient.getOrderHistory(userId), wrapped in try/catch treating any exception as an empty list; if empty, return an empty list immediately
2. From the history, take the most recent PLACED order (the last element, or explicitly the one with the latest createdAt if you parse it — either is acceptable for this version)
3. For each distinct productId in that order's items, call getRecommendationsForProduct(that productId) (reusing the method above) and merge the results
4. Dedupe the merged list by productId, and exclude any productId the user has already purchased anywhere in their order history (across ALL their orders, not just the most recent one)
5. Return the final deduped, filtered list — cap at RECOMMENDATION_COUNT total if the merge produced more
6. Never throw — any failure anywhere in this flow results in an empty list

CONTROLLER (package: com.ecommerce.recommendationservice.controller):

RecommendationController — base path /api/recommendations
- GET /api/recommendations/product/{productId} — calls getRecommendationsForProduct, returns 200 with List<ProductResponse> (empty list is a valid 200)
- GET /api/recommendations/user/{userId} — calls getRecommendationsForUser, returns 200 with List<ProductResponse> (empty list is a valid 200)
- No GlobalExceptionHandler / custom exceptions needed in this service — by design, RecommendationService never throws upward, every failure path already resolves to an empty list at the service layer. Do not add a @RestControllerAdvice class for this version, there is nothing for it to catch.

CONSTRAINTS:
- Do not add a database, an entity, or a repository anywhere — this service is stateless, full stop.
- Do not add caching of any kind — recomputing from live calls on every request is the deliberate v3 behavior.
- Do not add retry, circuit breaker, or timeout configuration on either Feign client — plain, unguarded calls wrapped in try/catch are correct for this version; resilience is deferred to v5.
- Do not add tie-breaking logic to Rule 1's sort — undefined order on equal counts is acceptable for v3.
- Do not add any endpoint beyond the two listed above.
- Do not add unit tests in this prompt unless you want to include the algorithm test now — if you do include one, keep it to a plain JUnit/Mockito test of RecommendationService's Rule 1 tallying logic with mocked Feign clients, @ExtendWith(MockitoExtension.class), no Spring context. If in doubt, leave tests out of this prompt entirely — testing checklist verification below is manual and sufficient for v3.
- Output every file in full: application.yml, all four DTOs, both Feign client interfaces, InternalSecretFilter, FilterConfig, FeignInternalSecretInterceptor, RecommendationService, RecommendationController, and RecommendationServiceApplication.java. Do NOT output pom.xml.

VERIFICATION (tell me how to confirm this works):
- Prerequisite: eureka-server, product-service, order-service must already be running and registered, and you should have at least one product with real PLACED-order history (from testing order-service's new containing-product endpoint in the previous prompt) plus at least one brand-new product with zero order history.
- Confirm recommendation-service registers with Eureka on startup.
- Send GET /api/recommendations/product/{id} directly to port 8084 (with the correct X-Internal-Secret header) for the product with order history — confirm it returns sensible co-occurrence results.
- Send the same request for the brand-new product with zero order history — confirm it falls back correctly to same-category results (Rule 2).
- Send GET /api/recommendations/user/{userId} for a user with at least one PLACED order — confirm results, and confirm none of the returned products were already purchased by that user.
- Send either request with NO X-Internal-Secret header — confirm 401, request never reaches the controller.
- Manually stop product-service (keep order-service up) and re-send GET /api/recommendations/product/{id} — confirm it returns an empty list, not a 500 or hung request. Restart product-service, stop order-service instead, repeat — confirm the same empty-list-not-500 behavior.
- With both services back up, cross-check a known-good recommendation result against manually calling order-service's containing-product endpoint directly, to confirm you're seeing genuine data and not a silently-rejected internal call (missing/wrong X-Internal-Secret on this service's own outbound calls would look identical to "no data").