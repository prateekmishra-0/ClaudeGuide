This is NOT a new project — you're continuing in the existing frontend-service project from v1 (Parts A and B). Do not run Spring Initializr again. No dependency changes are needed for this version — do not modify pom.xml at all, do not suggest any addition.

Existing files in this project (do not regenerate these from scratch — you are extending/modifying them):
- ProductServiceClient, UserServiceClient, OrderServiceClient Feign interfaces — MODIFIED below (one attribute change each), method signatures unchanged
- All DTOs (ProductResponse, ProductPageResponse, RegisterRequest, LoginRequest, LoginResponse, UserResponse, AddItemRequest, OrderItemResponse, OrderResponse) — unchanged, do not touch
- HomeController, ProductController — unchanged, do not touch
- OrderController — MODIFIED below (one new catch added, no new methods)
- AuthController, CartController — unchanged, do not touch. The Authorization header attachment happens entirely inside the new FeignAuthInterceptor bean below, which applies globally to every Feign call automatically — neither controller needs any code change to benefit from it.
- LoginRequiredInterceptor, GlobalModelAttributes, WebExceptionHandler — unchanged, do not touch
- All existing exceptions (ProductNotFoundException, EmailAlreadyExistsException, InvalidCredentialsException, EmptyCartException, InvalidRegistrationException) — unchanged, do not touch
- All existing templates (home, product-detail, register, login, cart, order-confirmation, orders, error, header fragment) — unchanged except cart.html, see below

MODIFY: ProductServiceClient, UserServiceClient, OrderServiceClient (package: com.ecommerce.frontendservice.client)
- Change the `name` attribute on all three @FeignClient annotations from the individual service's logical name to "api-gateway"
- Do not change any method signature, path, or return type — every existing path already includes the full /api/... prefix the gateway's routes expect

NEW: FeignAuthInterceptor (package: com.ecommerce.frontendservice.config)
- Class implementing feign.RequestInterceptor, annotated @Component
- Override apply(RequestTemplate template): read the current HttpServletRequest via RequestContextHolder.currentRequestAttributes() (cast to ServletRequestAttributes, call .getRequest()), get its HttpSession WITHOUT creating one if it doesn't exist (getSession(false)), and if the session exists AND has a non-null "sessionToken" attribute, add header "Authorization" with value "Bearer " + that token to the template
- If no session exists, or "sessionToken" is not present, add no header at all — do not throw, do not block the request
- This interceptor applies automatically to every Feign call made by any of the three clients above
- Do NOT add an X-Internal-Secret header anywhere in this class or anywhere else in this project — frontend-service never calls a backend service directly, it only ever calls api-gateway, and the gateway alone is responsible for attaching that header downstream. Attaching it here would be both unnecessary and architecturally wrong — it would suggest frontend-service is a trusted internal caller, which it is not; it's the public-facing tier.

RENAME SESSION ATTRIBUTE:
- No code change needed — AuthController already stores whatever userServiceClient.login() returns under the "sessionToken" key; the attribute name and storage location don't change, only the string's shape (JWT instead of UUID) changes, which is transparent to all existing code.

NEW EXCEPTION (package: com.ecommerce.frontendservice.exception):
- InsufficientStockException extends RuntimeException (constructor takes a String message)

MODIFY: OrderController (package: com.ecommerce.frontendservice.controller)
- POST /checkout — currently catches feign.FeignException.BadRequest (for EmptyCartException). Add a second catch: feign.FeignException.Conflict (order-service now returns 409 on insufficient stock) — re-throw as our own InsufficientStockException with a fixed message: "One or more items in your cart don't have enough stock available. Please review your cart and try again." Do not attempt to parse order-service's actual error message body.
- On InsufficientStockException, add "error" as a flash attribute with that message and redirect to "/cart" (same pattern as the existing EmptyCartException handling — a new, separate catch alongside it, not a replacement)

MODIFY: cart.html (src/main/resources/templates/)
- Add a check near the top: if a flash attribute "error" is present, display it prominently above the cart contents (now populated by two different failure cases — EmptyCartException and InsufficientStockException — both from the /checkout redirect; the template doesn't need to distinguish which)

CONSTRAINTS:
- Do not use JavaScript anywhere.
- Do not change LoginRequiredInterceptor's protected path patterns.
- Do not add a "remember me" or persistent-login feature.
- Do not modify HomeController, ProductController, LoginRequiredInterceptor, GlobalModelAttributes, WebExceptionHandler, or any DTO.
- Do not add the Authorization header manually inside any controller method — it must be attached automatically via FeignAuthInterceptor, for every authenticated Feign call.
- Do not add any internal-secret logic anywhere in this project, under any name — this service is explicitly outside that mechanism, as stated above.
- Output every file that actually changes, in full: the three modified Feign client interfaces, the new FeignAuthInterceptor, the new InsufficientStockException, the modified OrderController, and the modified cart.html. Do not regenerate or restate any DTO, HomeController, ProductController, CartController (unchanged), AuthController (unchanged), LoginRequiredInterceptor, GlobalModelAttributes, WebExceptionHandler, any other exception, or any other template.

VERIFICATION (tell me how to confirm this works):
- With eureka-server, api-gateway, product-service, user-service, and order-service all running (all four backend services should already have their InternalSecretFilter active, and api-gateway attaching the header, per the earlier prompts): browse the homepage and a product detail page while logged out — confirm both still work. Register, log in, confirm the header shows your name. Add an item to cart, view the cart — confirm this succeeds (proving the Authorization header reaches the gateway correctly, and the gateway's internal-secret attachment lets it reach order-service). Checkout normally with sufficient stock — confirm success. Manually create a scenario where checkout would exceed stock and confirm the cart page shows the specific "don't have enough stock" message rather than a generic error page. Log out and confirm attempting to visit /cart directly redirects to /login.
- As a final end-to-end sanity check on the whole internal-secret mechanism: temporarily stop api-gateway and try to use the frontend directly — every page should now fail (frontend's Feign calls target api-gateway by logical name, which is now unreachable), confirming there's no accidental direct path from frontend-service to any backend service that could have bypassed the mechanism entirely.