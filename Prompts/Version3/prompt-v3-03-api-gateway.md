This is NOT a new project — you're continuing in the existing api-gateway project from v2. Do not run Spring Initializr again. No dependency changes are needed for this version — do not modify pom.xml at all, do not suggest any addition.

Existing files in this project (do not regenerate these from scratch — you are extending/modifying them):
- ApiGatewayApplication.java — unchanged, do not touch
- JwtValidationFilter — EXTENDED below (whitelist condition only), no other logic changes

CONFIGURATION FILE CHANGE:
- File: src/main/resources/application.yaml
- Add ONE new route entry to the existing spring.cloud.gateway.server.webmvc.routes list — append it after order-route, do not create a new property block, do not create a new file:
    - id: recommendation-route
      uri: lb://recommendation-service
      predicates:
        - Path=/api/recommendations/**
- Keep every existing key exactly as it is — server.port, spring.application.name, the existing three routes (product-route, user-route, order-route), eureka.client.service-url.defaultZone, jwt.secret, internal.secret — do not remove, reorder, or modify anything already there.
- Do not use spring.cloud.gateway.mvc.routes or spring.cloud.gateway.routes — this remains spring.cloud.gateway.server.webmvc.routes exactly as established in v2, same webmvc variant, same reasoning: the older property paths silently bind to nothing on this classpath.

MODIFY: JwtValidationFilter (package: com.ecommerce.apigateway.security)
- Locate the existing whitelist check inside doFilterInternal (the block that currently skips straight to header-attachment for: POST /api/users/register, POST /api/users/login, and GET requests starting with /api/products or /api/categories)
- Add ONE new condition to that same whitelist check: method is GET and path starts with /api/recommendations/product — this makes GET /api/recommendations/product/{productId} publicly reachable with no JWT required, same treatment as the existing product/category GET whitelist entries
- Do NOT add /api/recommendations/user to the whitelist — that endpoint must remain in the authenticated set, requiring a valid Bearer token exactly like every other non-whitelisted route since v2. No code change is needed to make this happen — it falls into the existing default (non-whitelisted) branch automatically, since the whitelist is an explicit allow-list, not a deny-list.
- Do not change any other part of this filter — the 401 short-circuit logic, the JWT parsing/validation logic, the X-Internal-Secret/X-User-Id/X-User-Email header-attachment logic via the HttpServletRequestWrapper, and getOrder() all stay exactly as they are from v2.

CONSTRAINTS:
- Do not add role-based checks in this filter — that's v5.
- Do not add retry, circuit breaker, or fallback logic to any route — that's v5.
- Do not add rate limiting or any other Gateway filter beyond what already exists.
- Do not add a custom /fallback endpoint — that's v5.
- Do not touch the internal-secret attachment logic — it already applies unconditionally to every request reaching step 6, including this new route, with zero changes needed.
- Do not modify ApiGatewayApplication.java.
- Output only the files that actually change: application.yaml (full file) and JwtValidationFilter.java (full file, including its nested/private HttpServletRequestWrapper class exactly as it already exists from v2 — restate it unchanged, do not regenerate its internals differently). Do NOT output pom.xml.

VERIFICATION (tell me how to confirm this works):
- On startup, confirm the console logs show all four routes now mapped (product-route, user-route, order-route, recommendation-route) — if recommendation-route is missing from the startup logs, re-check the exact indentation under spring.cloud.gateway.server.webmvc.routes before looking anywhere else.
- With eureka-server, product-service, order-service, and recommendation-service all running and registered: send GET http://localhost:8000/api/recommendations/product/{id} with NO Authorization header — expect 200 with recommendation data, confirming the new whitelist entry works and recommendation-service receives X-Internal-Secret correctly via the gateway.
- Send GET http://localhost:8000/api/recommendations/user/{userId} with NO Authorization header — expect 401 directly from the gateway, confirming this endpoint correctly stayed OUT of the whitelist.
- Repeat the user-level request WITH a valid "Authorization: Bearer <token>" header — expect success, and confirm recommendation-service's request logs show X-User-Id and X-User-Email were forwarded (even though recommendation-service's own controller doesn't currently use them for anything in v3 — the header-attachment logic doesn't know or care which downstream service will use them).
- Confirm the three pre-existing routes (product, user, order) still behave exactly as they did in v2 — this change should be purely additive with zero regression.
- No unit tests are specified for api-gateway in this version — manual verification via the steps above is sufficient, same as v2.