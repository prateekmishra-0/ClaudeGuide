This is NOT a new project — you're continuing in the existing product-service project from v1. Do not run Spring Initializr again. No dependency changes are needed for this version — do not modify pom.xml at all, do not suggest any addition. Everything needed for this version's change (a plain servlet filter) is already available via the existing Spring Web dependency.

Existing files in this project (do not regenerate these from scratch — you are extending/modifying them):
- Category, Product entities — unchanged, do not touch
- CategoryRepository, ProductRepository — unchanged, do not touch
- CategoryController, ProductController — unchanged, do not touch, no endpoint logic changes in this version
- All four custom exceptions (ProductNotFoundException, CategoryNotFoundException, InsufficientStockException, CategoryAlreadyExistsException) — unchanged, do not touch
- GlobalExceptionHandler — unchanged, do not touch, no new exception types are introduced in this version
- ProductControllerTest, CategoryControllerTest — unchanged, do not touch

CONFIGURATION FILE CHANGE:
- File: src/main/resources/application.yaml
- Add this one new key (keep every existing key exactly as it is — datasource, JPA, Eureka, port, etc. — do not remove or modify anything already there):
  internal.secret: <PASTE_YOUR_INTERNAL_SECRET_HERE> — this exact string must be the SAME value already placed in api-gateway's and user-service's application.yml/application.yaml files, and the same value that will also go into order-service's config in a separate prompt. Do not generate a new value.

NEW CLASS (package: com.ecommerce.productservice.security):
- Class: InternalSecretFilter, extends OncePerRequestFilter, annotated @Component
- Reads internal.secret from application.yaml via @Value
- In doFilterInternal(request, response, filterChain):
    1. Read header "X-Internal-Secret" from the request
    2. If missing OR does not exactly equal the configured internal.secret value, write HTTP status 401 directly to the response, set content type application/json, write body {"message": "Forbidden — direct access not permitted"}, and RETURN without calling filterChain.doFilter() — the request must never reach any controller
    3. If it matches, call filterChain.doFilter(request, response) to let the request proceed normally
- This filter applies to EVERY endpoint in this service with NO exceptions — including GET /api/products and GET /api/categories, which remain publicly reachable to end-users only via api-gateway (which attaches this header on their behalf), never directly. There is no whitelist in this filter.
- This filter has nothing to do with JWTs or user identity — product-service never sees a JWT at all, in this or any prior version. This is a simple, unconditional string-equality check against one fixed header value, unrelated to any user-level authentication.

NEW CLASS (package: com.ecommerce.productservice.config):
- Class: FilterConfig, annotated @Configuration
- A @Bean method of type FilterRegistrationBean<InternalSecretFilter>, registering InternalSecretFilter with .setOrder(Ordered.HIGHEST_PRECEDENCE) explicitly, so it runs before anything else touches the request — do not rely on @Order on the filter class alone

CONSTRAINTS:
- Do not modify any entity, repository, controller, exception, or existing test — this version's change is confined entirely to the two new files (InternalSecretFilter, FilterConfig) and the one new application.yaml key.
- Do not add any test class for InternalSecretFilter — the existing @WebMvcTest test classes (ProductControllerTest, CategoryControllerTest) do not load servlet filters registered via FilterRegistrationBean by default, so adding a test here would be misleading; manual verification (below) is sufficient for this version.
- Do not touch pom.xml — no dependency is needed for this change, everything required is already present via Spring Web.
- Output only the files that actually change: application.yaml (full file), the new InternalSecretFilter class (full file), and the new FilterConfig class (full file). Do not regenerate or restate any entity, repository, controller, exception, GlobalExceptionHandler, or any existing test class.

VERIFICATION (tell me how to confirm this works):
- With product-service running standalone (no other service needs to be up for this check): send GET /api/products directly to port 8081 with NO X-Internal-Secret header — confirm 401 with the "Forbidden — direct access not permitted" message, even though this would otherwise be a perfectly valid, publicly-readable request. Repeat the same request WITH the correct X-Internal-Secret header — confirm it now returns the product list normally (200).
- Repeat both checks against a write endpoint too, e.g. POST /api/categories — confirm the same 401-without-header / success-with-header pattern holds, proving the filter applies uniformly regardless of endpoint or HTTP method.
- Once api-gateway is also built and running: confirm GET http://localhost:8000/api/products (through the gateway) still succeeds with no Authorization header from the client — this proves the gateway is correctly attaching X-Internal-Secret on the client's behalf, and product-service is correctly accepting it.
- No Maven test command is needed for this prompt — no new automated tests were added.