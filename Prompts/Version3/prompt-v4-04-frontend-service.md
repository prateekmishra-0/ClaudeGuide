This is NOT a new project — you're continuing in the existing frontend-service project from v1 (Parts A and B) and v2. Do not run Spring Initializr again. No dependency changes are needed for this version — do not modify pom.xml at all, do not suggest any addition.

Existing files in this project (do not regenerate these from scratch — you are extending/modifying them):
- ProductServiceClient, UserServiceClient, OrderServiceClient Feign interfaces — unchanged, do not touch
- FeignAuthInterceptor — unchanged, do not touch; it applies automatically to any new Feign client too, since it's a globally-picked-up RequestInterceptor bean, no per-client wiring needed
- All existing DTOs — unchanged, do not touch
- ProductController — unchanged, do not touch
- HomeController — MODIFIED below (one new conditional call), no other logic changes
- AuthController, CartController, OrderController — unchanged, do not touch
- LoginRequiredInterceptor, GlobalModelAttributes, WebExceptionHandler — unchanged, do not touch
- All existing exceptions — unchanged, do not touch
- product-detail.html — MODIFIED below (one new section added)
- home.html — MODIFIED below (one new conditional section added)
- All other existing templates (register, login, cart, order-confirmation, orders, error, header fragment) — unchanged, do not touch

No configuration file change in this version — application.yaml stays exactly as it is from v2.

NEW DTO (package: com.ecommerce.frontendservice.dto):
- ProductResponse already exists from Part A — reuse it as-is, do not create a duplicate or a differently-named copy. If for some reason the existing ProductResponse doesn't already have every field recommendation-service returns, use exactly the existing fields as-is (id, name, description, price, stockQuantity, categoryId) — do not add new fields to it.

NEW FEIGN CLIENT (package: com.ecommerce.frontendservice.client):

Interface: RecommendationServiceClient, annotated @FeignClient(name = "api-gateway") — same pattern v2 established for the other three clients, targeting the gateway by its logical name, not recommendation-service directly
- @GetMapping("/api/recommendations/product/{productId}") List<ProductResponse> getProductRecommendations(@PathVariable("productId") Long productId)
- @GetMapping("/api/recommendations/user/{userId}") List<ProductResponse> getUserRecommendations(@PathVariable("userId") Long userId)
- Purely declarative, no implementation body.
- Error handling: do not write a custom ErrorDecoder. Wrap both call sites (in ProductController and HomeController respectively) in try/catch treating ANY exception as an empty list — recommendation-service itself already fails open and returns empty lists rather than errors, but a network blip or a stopped recommendation-service could still surface as a FeignException here, and a missing recommendation widget must never break the page it's on.

MODIFY: ProductController (package: com.ecommerce.frontendservice.controller)
- GET /products/{id} — after the existing product-lookup logic (unchanged), add a call to recommendationServiceClient.getProductRecommendations(id), wrapped in try/catch as described above (on any exception, use an empty list), add the result to the Model under attribute name "recommendations", then proceed to return view name "product-detail" exactly as before
- This call happens for EVERY visitor, logged in or not — do not gate it behind a login check, product-detail.html is not a login-gated page.

MODIFY: HomeController (package: com.ecommerce.frontendservice.controller)
- GET / — after the existing product-listing logic (unchanged), check whether the user is logged in by reading HttpSession attribute "userId" (inject HttpSession as a method parameter, same access pattern used elsewhere in this project, e.g. GlobalModelAttributes)
    - If logged in: call recommendationServiceClient.getUserRecommendations(userId), wrapped in try/catch as described above (on any exception, use an empty list), add the result to the Model under attribute name "userRecommendations"
    - If not logged in: do not call the endpoint at all — there is no order history to work from, and calling it anyway would just return an empty list for no benefit. Simply don't add "userRecommendations" to the Model in this case (or add an empty list — either is fine, the template below checks for null/empty either way).
- Return view name "home" exactly as before.

MODIFY: product-detail.html
- Add a new section below the existing product detail content (and below the add-to-cart form from Part B): a "You might also like" heading followed by a grid/list rendering the "recommendations" model attribute — for each recommended product, show name, price, and a link to /products/{id}, same visual pattern as the existing product cards on home.html
- If "recommendations" is empty, do not show the heading or an empty grid at all — the section should simply not render, not show a "no recommendations" message (an empty section is a normal, unremarkable case here, not worth calling out to the visitor)

MODIFY: home.html
- Add a new section, rendered ONLY if "userRecommendations" is present and non-empty (th:if): a "Recommended for you" heading followed by a grid/list of those products, same card pattern as the existing product grid, each linking to /products/{id}
- Logged-out visitors see the plain existing v1 product grid with no change at all — this section must not render for them under any circumstance
- Place this section above or below the main product grid, your choice — either is fine, this project has no strict layout requirement beyond what's already established

CONSTRAINTS:
- Do not use JavaScript anywhere, including for this new content — plain server-rendered HTML only, same as every other page in this project.
- Do not add the Authorization header manually anywhere — FeignAuthInterceptor already attaches it automatically to every Feign call made by any client targeting "api-gateway", including this new one, with zero additional code needed.
- Do not add any internal-secret logic anywhere in this project — frontend-service never calls a backend service directly, that constraint from v2 is unchanged and applies to this new client too.
- Do not modify AuthController, CartController, OrderController, LoginRequiredInterceptor, GlobalModelAttributes, WebExceptionHandler, or any existing exception.
- Do not add a new exception type for recommendation failures — per the fail-open principle, a failed recommendation call is caught locally at the call site and degrades to an empty list, it never needs to reach WebExceptionHandler at all.
- Do not add caching of the recommendation results anywhere in this service.
- Output every file that actually changes, in full: the new RecommendationServiceClient, the modified ProductController, the modified HomeController, the modified product-detail.html, and the modified home.html. Do not regenerate or restate any DTO, any unchanged controller, any unchanged template, LoginRequiredInterceptor, GlobalModelAttributes, WebExceptionHandler, FeignAuthInterceptor, or any exception.

VERIFICATION (tell me how to confirm this works):
- With eureka-server, api-gateway, product-service, user-service, order-service, and recommendation-service all running: visit a product detail page while logged OUT — confirm the "You might also like" section renders with real recommended products (proving the public whitelist entry from the previous prompt works end-to-end through the full stack, not just directly against the gateway).
- Visit the homepage while logged OUT — confirm no "Recommended for you" section appears anywhere, and the page otherwise looks identical to v1/v2.
- Log in as a user with at least one PLACED order, visit the homepage — confirm the "Recommended for you" section now appears with sensible results.
- Visit a brand-new product's detail page (zero order history) — confirm the "You might also like" section still renders, showing same-category fallback results rather than an empty or broken section.
- Temporarily stop recommendation-service entirely, then revisit both a product detail page and the (logged-in) homepage — confirm both pages still render completely normally, simply without their recommendation sections, and confirm no error page or stack trace appears anywhere. Restart recommendation-service afterward and confirm the sections come back.