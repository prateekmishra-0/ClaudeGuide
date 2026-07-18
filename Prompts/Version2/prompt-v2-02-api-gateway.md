I have already generated this Spring Boot project via Spring Initializr, with one manual addition to pom.xml — do not modify, add, or remove anything else in pom.xml, and do not change the Spring Boot or Spring Cloud version:

- Spring Boot 4.1.0, Spring Cloud release train 2025.1.2 (already configured in pom.xml)
- Group: com.ecommerce, Artifact: api-gateway, Package: com.ecommerce.apigateway
- Dependencies present: spring-cloud-starter-gateway-server-webmvc, spring-cloud-starter-netflix-eureka-client, spring-cloud-starter-loadbalancer, and three manually added jjwt dependencies (jjwt-api, jjwt-impl, jjwt-jackson, version 0.12.6)
- Configuration format: application.yml (already exists, currently empty/default)
- IMPORTANT — this is the SERVLET/MVC variant of Spring Cloud Gateway (spring-cloud-starter-gateway-server-webmvc), NOT the reactive WebFlux variant. This runs on the same Tomcat/servlet stack as every other service in this project. Do NOT use GlobalFilter, GatewayFilterChain, Mono, ServerWebExchange, or anything from the org.springframework.web.reactive or reactor.core packages anywhere in this project — those belong to the reactive variant and are not on this project's classpath at all. Write the custom filter as a plain jakarta.servlet Filter (specifically OncePerRequestFilter), the exact same pattern already used in product-service and order-service.
- CRITICAL — routes property path: as of Spring Cloud release train 2025.0.0 and later, the property-based route configuration for this artifact lives under spring.cloud.gateway.server.webmvc.routes, NOT spring.cloud.gateway.mvc.routes (that older key belonged to the pre-GA incubating module and is not bound by spring-cloud-starter-gateway-server-webmvc — it will not register any routes, will not throw an error, and will not log a warning; it will just silently do nothing). Use spring.cloud.gateway.server.webmvc.routes exactly as shown in the CONFIGURATION FILE section below. Do not use any other variant of this property path.

Your task is ONLY to write the application logic and configuration on top of this existing project. Do not touch pom.xml. Do not suggest adding spring-boot-starter-web, spring-cloud-starter-gateway-server-webflux, or any other dependency beyond what's listed above.

MAIN APPLICATION CLASS:
- File: src/main/java/com/ecommerce/apigateway/ApiGatewayApplication.java
- Annotations: @SpringBootApplication, @EnableDiscoveryClient

CONFIGURATION FILE:
- File: src/main/resources/application.yaml
- Reproduce this structure exactly, substituting the two placeholder values:

server:
port: 8000
spring:
application:
name: api-gateway
cloud:
gateway:
server:
webmvc:
routes:
- id: product-route
uri: lb://product-service
predicates:
- Path=/api/products/**,/api/categories/**
- id: user-route
uri: lb://user-service
predicates:
- Path=/api/users/**
- id: order-route
uri: lb://order-service
predicates:
- Path=/api/orders/**
eureka:
client:
service-url:
defaultZone: http://localhost:8761/eureka
jwt:
secret: 0bXHZaF1C3tnxFtxM1IcLYCxCQCv1z1a4eX2KKkGlck=
internal:
secret: arhjYkqjrB6INMZvZjOApL6jk0b6LWJVnsEYl29kMY0=

- jwt.secret must be the EXACT same string already placed in user-service's application.yaml jwt.secret key.
- internal.secret must be a SEPARATE, DIFFERENT string from jwt.secret, and must be the EXACT same value that will also be placed in product-service's and order-service's application.yaml files in separate prompts.
- Do not add, remove, or rename any route id, predicate, or top-level key beyond what's shown above.

NEW CLASS (package: com.ecommerce.apigateway.security):
- Class: JwtValidationFilter, extends OncePerRequestFilter, implements Ordered, annotated @Component
- Override getOrder() to return Ordered.HIGHEST_PRECEDENCE, so this filter runs before the gateway's own routing/forwarding logic
- Override doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain):
    1. Read the request's path (request.getRequestURI()) and HTTP method (request.getMethod())
    2. Whitelist check — skip straight to step 6 below (no JWT check, but the internal-secret attachment in step 6 still applies) if ANY of these match:
        - method is POST and path equals exactly /api/users/register
        - method is POST and path equals exactly /api/users/login
        - method is GET and path starts with /api/products or /api/categories
    3. For everything else: read the Authorization header; if missing or doesn't start with "Bearer ", short-circuit — set response status to 401, set content type application/json, write body {"message": "Missing or invalid Authorization header"} directly to response.getWriter(), and RETURN without calling filterChain.doFilter() — the request must never reach routing/proxying at all
    4. If present, extract the token (strip "Bearer " prefix), attempt to parse and validate it using the jjwt 0.12.x API (Jwts.parser().verifyWith(secretKey).build().parseSignedClaims(token)) with a SecretKey built from jwt.secret (Keys.hmacShaKeyFor() on UTF-8 bytes)
    5. If parsing/validation throws any jjwt exception — same 401 short-circuit as step 3, message "Invalid or expired token", do not proceed to step 6
    6. Since this is a plain servlet filter, you cannot mutate an incoming HttpServletRequest's headers directly (unlike the reactive variant's exchange.mutate() pattern) — instead, wrap the request in a custom HttpServletRequestWrapper subclass (e.g. a small private/nested class HeaderMapRequestWrapper) that overrides getHeader(), getHeaders(), and getHeaderNames() to layer additional headers on top of the original request's headers, without modifying the original. Always add X-Internal-Secret (from the internal.secret config value) via this wrapper — on EVERY request that reaches this point, whitelisted or not. If the request had a valid JWT (i.e. it went through steps 3-5 successfully), also add X-User-Id (the "sub" claim value) and X-User-Email (the "email" claim value) via the same wrapper. If the request was whitelisted (no JWT involved), add ONLY X-Internal-Secret, not the two user-identity headers.
    7. Call filterChain.doFilter(wrappedRequest, response), passing the WRAPPED request (not the original), so the downstream gateway routing logic — and ultimately the proxied backend service — sees the added headers

- Read jwt.secret and internal.secret via @Value in this class

CONSTRAINTS:
- Do not use GlobalFilter, GatewayFilterChain, Mono, ServerWebExchange, or any reactive type anywhere — this project has no WebFlux dependency on its classpath and none of that code will compile.
- Do not add role-based checks in this filter.
- Do not add retry, circuit breaker, or fallback logic to any route.
- Do not add rate limiting or any other Gateway filter beyond the one described above.
- Do not add a custom /fallback endpoint.
- X-Internal-Secret must be attached on every single request that reaches step 6, with zero exceptions — including whitelisted public GET/register/login calls.
- Do not use spring.cloud.gateway.mvc.routes, spring.cloud.gateway.routes, or any property path other than spring.cloud.gateway.server.webmvc.routes for route configuration.
- Output every file in full: application.yml, ApiGatewayApplication.java, and JwtValidationFilter.java (including the nested/private HttpServletRequestWrapper class within it, or as its own small file if cleaner — your choice, just output whichever files you create in full). Do NOT output pom.xml.

VERIFICATION (tell me how to confirm this works):
- On startup, confirm the console logs show all three routes being mapped (look for RouterFunctionMapping or similar log lines referencing product-route, user-route, order-route — if you see none of these, the routes property didn't bind and nothing will work).
- With eureka-server, user-service, product-service, and order-service all already running and registered: confirm the gateway logs show all three routes resolved on startup. Test: GET http://localhost:8000/api/products with no Authorization header — expect 200 (public whitelist), and confirm on product-service's side that X-Internal-Secret was still attached even though no JWT was required. POST http://localhost:8000/api/users/login with valid credentials, no Authorization header — expect 200 with a token. Take that token, call GET http://localhost:8000/api/orders/history/1 with no Authorization header — expect 401 directly from the gateway (never reaches order-service). Repeat WITH "Authorization: Bearer <token>" — expect success, and confirm order-service receives X-Internal-Secret, X-User-Id, and X-User-Email all three this time. Finally, manually alter one character of a valid token and confirm 401 ("Invalid or expired token").
- If any whitelisted GET route (e.g. /api/products) still returns the generic Spring Boot 404 shape {"timestamp","status":404,"error":"Not Found",...} rather than actual product-service data, that means the routes still aren't registered — re-check the exact indentation and key path of spring.cloud.gateway.server.webmvc.routes in application.yml before looking anywhere else.
- No unit tests are specified for this service in this version — manual verification via the steps above is sufficient.