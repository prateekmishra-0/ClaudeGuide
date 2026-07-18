I have already generated this Spring Boot project via Spring Initializr, with one manual addition to pom.xml — do not modify, add, or remove anything else in pom.xml, and do not change the Spring Boot or Spring Cloud version:

- Spring Boot 4.1.0, Spring Cloud release train 2025.1.2 (already configured in pom.xml)
- Group: com.ecommerce, Artifact: api-gateway, Package: com.ecommerce.apigateway
- Dependencies present: spring-cloud-starter-gateway, spring-cloud-starter-netflix-eureka-client, and three manually added jjwt dependencies (jjwt-api, jjwt-impl, jjwt-jackson, version 0.12.6)
- Configuration format: application.yml (already exists, currently empty/default)
- This is a Spring Cloud Gateway project — it runs on the reactive (WebFlux) stack, NOT the servlet stack. Do not use anything from javax.servlet/jakarta.servlet, HttpServletRequest, or any Filter/OncePerRequestFilter pattern anywhere in this project — those are servlet-stack constructs and will not compile or function correctly here. Use Gateway's own GlobalFilter and GatewayFilterChain types throughout.

Your task is ONLY to write the application logic and configuration on top of this existing project. Do not touch pom.xml. Do not suggest adding spring-boot-starter-web or any other dependency beyond what's listed above.

MAIN APPLICATION CLASS:
- File: src/main/java/com/ecommerce/apigateway/ApiGatewayApplication.java
- Annotations: @SpringBootApplication, @EnableDiscoveryClient

CONFIGURATION FILE:
- File: src/main/resources/application.yml
- server.port: 8000
- spring.application.name: api-gateway
- eureka.client.service-url.defaultZone: http://localhost:8761/eureka
- jwt.secret: <PASTE_YOUR_JWT_SECRET_HERE> — must be the EXACT same string already placed in user-service's application.yaml jwt.secret key
- internal.secret: <PASTE_YOUR_INTERNAL_SECRET_HERE> — a SEPARATE, DIFFERENT string from jwt.secret; must be the EXACT same value that will also be placed in product-service's and order-service's application.yaml files in separate prompts; do not confuse the two secrets
- spring.cloud.gateway.routes — define three routes here in YAML (not Java RouteLocator beans):
    1. id: product-route, uri: lb://product-service, predicates: Path=/api/products/**,/api/categories/**
    2. id: user-route, uri: lb://user-service, predicates: Path=/api/users/**
    3. id: order-route, uri: lb://order-service, predicates: Path=/api/orders/**

NEW CLASS (package: com.ecommerce.apigateway.security):
- Class: JwtValidationFilter, implements GlobalFilter and Ordered
- Override getOrder() to return a high-priority value (e.g. -1) so it runs before routing
- Override filter(ServerWebExchange exchange, GatewayFilterChain chain):
    1. Read the incoming request's path and HTTP method from exchange.getRequest()
    2. Whitelist check — skip straight to step 6 below (no JWT check, but the internal-secret attachment in step 6 still applies) if ANY of these match:
        - method is POST and path equals exactly /api/users/register
        - method is POST and path equals exactly /api/users/login
        - method is GET and path starts with /api/products or /api/categories
    3. For everything else: read the Authorization header; if missing or doesn't start with "Bearer ", short-circuit — set exchange.getResponse() status to 401, write a small JSON body {"message": "Missing or invalid Authorization header"}, return Mono.empty() (do not call chain.filter, do not proceed to step 6 — an unauthenticated request should not reach any downstream service at all, not even with the internal secret attached)
    4. If present, extract the token (strip "Bearer " prefix), attempt to parse and validate it using the jjwt 0.12.x API (Jwts.parser().verifyWith(secretKey).build().parseSignedClaims(token)) with a SecretKey built from jwt.secret (Keys.hmacShaKeyFor() on UTF-8 bytes)
    5. If parsing/validation throws any jjwt exception — same 401 short-circuit as step 3, message "Invalid or expired token", do not proceed to step 6
    6. Build the outgoing request: always add header X-Internal-Secret with the value from the internal.secret config property — this happens on EVERY request that reaches this point, whitelisted or not, since the internal secret proves gateway origin regardless of whether a user JWT was required for that particular path. If the request had a valid JWT (i.e. it went through steps 3-5 successfully), also add X-User-Id (the "sub" claim value) and X-User-Email (the "email" claim value). If the request was whitelisted (no JWT involved at all), do NOT add X-User-Id or X-User-Email — only X-Internal-Secret.
    7. Use exchange.mutate().request(builder -> builder.header(...)).build() to attach whichever headers apply, then call chain.filter() with the mutated exchange

- Read jwt.secret and internal.secret via @Value in this class

CONSTRAINTS:
- Do not re-implement this as a Servlet Filter, OncePerRequestFilter, or anything from spring-security.
- Do not add role-based checks in this filter.
- Do not add retry, circuit breaker, or fallback logic to any route.
- Do not add rate limiting or any other Gateway filter beyond the one described above.
- Do not add a custom /fallback endpoint.
- X-Internal-Secret must be attached on every single request that reaches step 6, with zero exceptions — including whitelisted public GET/register/login calls. Do not make this conditional on anything other than "did the request pass the JWT check or was it whitelisted" (i.e. did it NOT get short-circuited in steps 3/5).
- Output every file in full: application.yml, ApiGatewayApplication.java, and JwtValidationFilter.java. Do NOT output pom.xml.

VERIFICATION (tell me how to confirm this works):
- With eureka-server, user-service, product-service, and order-service all already running and registered (product-service and order-service should already have their own InternalSecretFilter active if built in this order — otherwise this step will 401 at the receiving service instead, which is also a valid way to confirm the header is being sent): confirm the gateway dashboard/logs show all three routes resolved. Test: GET http://localhost:8000/api/products with no Authorization header — expect 200 (public whitelist), and confirm on product-service's side (logs, or by temporarily removing its filter to inspect) that X-Internal-Secret was still attached even though no JWT was required. POST http://localhost:8000/api/users/login with valid credentials, no Authorization header — expect 200 with a token. Take that token, call GET http://localhost:8000/api/orders/history/1 with no Authorization header — expect 401 directly from the gateway (never reaches order-service). Repeat WITH "Authorization: Bearer <token>" — expect success, and confirm order-service receives X-Internal-Secret, X-User-Id, and X-User-Email all three this time. Finally, manually alter one character of a valid token and confirm 401 ("Invalid or expired token").
- No unit tests are specified for this service in this version — manual verification via the steps above is sufficient.