This is NOT a new project — you're continuing in the existing api-gateway project from v2/v3. Do not run Spring Initializr again. No dependency changes are needed for this version — do not modify pom.xml at all, do not suggest any addition.

Existing files in this project (do not regenerate these from scratch — you are extending/modifying them):
- ApiGatewayApplication.java — unchanged, do not touch
- JwtValidationFilter — unchanged, do not touch. No new whitelist entries are needed in this version — both new routes below require a valid JWT, same as every non-whitelisted route since v2, and no code change is needed to make that happen since the whitelist is an explicit allow-list, not a deny-list.

CONFIGURATION FILE CHANGE:
- File: src/main/resources/application.yaml
- Add TWO new route entries to the existing spring.cloud.gateway.server.webmvc.routes list — append them after recommendation-route, do not create a new property block, do not create a new file:
    - id: payment-route
      uri: lb://payment-service
      predicates:
        - Path=/api/payments/**
    - id: notification-route
      uri: lb://notification-service
      predicates:
        - Path=/api/notifications/**
- Keep every existing key exactly as it is — server.port, spring.application.name, the existing four routes (product-route, user-route, order-route, recommendation-route), eureka.client.service-url.defaultZone, jwt.secret, internal.secret — do not remove, reorder, or modify anything already there.
- Do not use spring.cloud.gateway.mvc.routes or spring.cloud.gateway.routes — this remains spring.cloud.gateway.server.webmvc.routes exactly as established in v2/v3, same webmvc variant, same reasoning: the older property paths silently bind to nothing on this classpath.

CONSTRAINTS:
- Do not add either new route to JwtValidationFilter's whitelist — unlike v3's recommendation-route (which had one public GET path), both payment-service and notification-service are exclusively called internally by order-service via direct Eureka resolution in this version's design; neither is meant to be hit anonymously by an end user through the gateway. Both routes should require a valid Bearer token, same as the default (non-whitelisted) behavior every other non-whitelisted route already has — no code change accomplishes this, it's the existing default.
- Do not add role-based checks to either route — that's v5.
- Do not add retry, circuit breaker, or fallback logic to any route — that's v5.
- Do not add rate limiting or any other Gateway filter beyond what already exists.
- Do not add a custom /fallback endpoint — that's v5.
- Do not touch the internal-secret attachment logic — it already applies unconditionally to every request reaching that step, including these two new routes, with zero changes needed.
- Do not modify ApiGatewayApplication.java or JwtValidationFilter.java in any way.
- Output only the file that actually changes: application.yaml (full file). Do NOT output pom.xml, ApiGatewayApplication.java, or JwtValidationFilter.java — none of them change in this prompt.

VERIFICATION (tell me how to confirm this works):
- On startup, confirm the console logs show all six routes now mapped (product-route, user-route, order-route, recommendation-route, payment-route, notification-route) — if either new route is missing from the startup logs, re-check the exact indentation under spring.cloud.gateway.server.webmvc.routes before looking anywhere else.
- With eureka-server, payment-service, and notification-service all running and registered: send GET http://localhost:8000/api/payments/order/{orderId} (for a real orderId with a payment on record) with NO Authorization header — expect 401 directly from the gateway, confirming this route correctly stayed OUT of the whitelist.
- Repeat the same request WITH a valid "Authorization: Bearer <token>" header — expect success, confirming the route reaches payment-service correctly and X-Internal-Secret is attached automatically.
- Send POST http://localhost:8000/api/notifications with a valid Bearer token and a body like {"orderId": 1, "email": "your.email@example.com", "message": "manual gateway test"} — confirm a real email arrives, proving this route works end-to-end through the full stack, not just directly against notification-service on port 8086.
- Confirm the four pre-existing routes (product, user, order, recommendation) still behave exactly as they did in v3 — this change should be purely additive with zero regression.
- No unit tests are specified for api-gateway in this version — manual verification via the steps above is sufficient, same as v2 and v3.