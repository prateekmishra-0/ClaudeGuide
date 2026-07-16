# PROMPT 6 — frontend-service, Part B (auth + cart + checkout + history)

> Paste everything in the code block below directly into Amazon Q as a single prompt.
> This extends the SAME frontend-service project from Prompt 5 — do not create a new project, add these files into the existing one. user-service and order-service must already be built for the verification step, but this prompt itself doesn't need them running to generate correctly.
> This is the largest single prompt in v1. If the response gets cut off, tell me exactly which files were completed and which weren't, and I'll write a 6b covering only what's missing — don't re-paste the whole thing.

```
Extend the existing frontend-service Spring Boot + Thymeleaf project (already has: FrontendServiceApplication with @EnableFeignClients already present, ProductServiceClient Feign interface, HomeController, ProductController, home.html, product-detail.html, error.html, a header fragment, style.css). Add authentication, cart, checkout, and order history on top of it. Do not modify the existing product browsing files except product-detail.html and the header fragment, as noted below.

NO JAVASCRIPT — same constraint as Part A. All new interactivity is plain HTML forms with standard POST/GET.

NEW DTOs (package: com.ecommerce.frontendservice.dto):
- RegisterRequest: name (String), email (String), password (String)
- LoginRequest: email (String), password (String)
- LoginResponse: token (String)
- UserResponse: id (Long), name (String), email (String), role (String)
- AddItemRequest: productId (Long), quantity (Integer)
- OrderItemResponse: id (Long), productId (Long), productNameSnapshot (String), priceSnapshot (BigDecimal), quantity (Integer)
- OrderResponse: id (Long), userId (Long), status (String), items (List<OrderItemResponse>), createdAt (String)

NEW FEIGN CLIENTS (package: com.ecommerce.frontendservice.client):

Interface: UserServiceClient, annotated @FeignClient(name = "user-service")
- @PostMapping("/api/users/register") UserResponse register(@RequestBody RegisterRequest request)
- @PostMapping("/api/users/login") LoginResponse login(@RequestBody LoginRequest request)
- @GetMapping("/api/users/me") UserResponse getCurrentUser(@RequestHeader("X-Session-Token") String token)
- Purely declarative — no implementation body. Error handling happens at the call site (see AuthController below): catch feign.FeignException.Conflict from register (user-service returns 409 on duplicate email) and re-throw as our own EmailAlreadyExistsException; catch feign.FeignException.Unauthorized from login (user-service returns 401 on bad credentials) and re-throw as our own InvalidCredentialsException.
- Also catch feign.FeignException.BadRequest from register (user-service returns 400 when @Valid field validation fails, e.g. an invalid name or a too-short password) and re-throw as our own InvalidRegistrationException — do not attempt to parse or deserialize user-service's field-error JSON body; a fixed, generic message is sufficient for this version.

Interface: OrderServiceClient, annotated @FeignClient(name = "order-service")
- @GetMapping("/api/orders/cart/{userId}") OrderResponse getCart(@PathVariable("userId") Long userId)
- @PostMapping("/api/orders/cart/{userId}/items") OrderResponse addItem(@PathVariable("userId") Long userId, @RequestBody AddItemRequest request)
- @DeleteMapping("/api/orders/cart/{userId}/items/{itemId}") void removeItem(@PathVariable("userId") Long userId, @PathVariable("itemId") Long itemId) — note: the path is /api/orders/cart/{userId}/items/{itemId}, userId is required in the path even though itemId alone identifies the row
- @PostMapping("/api/orders/checkout/{userId}") OrderResponse checkout(@PathVariable("userId") Long userId) — catch feign.FeignException.BadRequest at the call site (order-service returns 400 on an empty cart) and re-throw as our own EmptyCartException
- @GetMapping("/api/orders/history/{userId}") List<OrderResponse> getOrderHistory(@PathVariable("userId") Long userId)

NEW EXCEPTIONS (package: com.ecommerce.frontendservice.exception):
- EmailAlreadyExistsException extends RuntimeException (constructor takes a String message)
- InvalidRegistrationException extends RuntimeException (constructor takes a String message)
- InvalidCredentialsException extends RuntimeException (constructor takes a String message)
- EmptyCartException extends RuntimeException (constructor takes a String message)

SESSION HANDLING:
- On successful login, store three attributes in HttpSession: "sessionToken" (String), "userId" (Long), "userName" (String) — userId and userName come from calling userServiceClient.getCurrentUser(token) immediately after login succeeds, so the session has the resolved user details, not just the raw token
- Logout simply invalidates the HttpSession

LOGIN-REQUIRED INTERCEPTOR (package: com.ecommerce.frontendservice.config):
- Class: LoginRequiredInterceptor implementing HandlerInterceptor — in preHandle, check if HttpSession attribute "userId" is present; if not, redirect to "/login" and return false; if present, return true
- Register this interceptor via a WebMvcConfigurer implementation, applied to path patterns: /cart/**, /checkout, /orders/**
- Do NOT apply it to /, /products/**, /register, /login — those remain accessible without login

GLOBAL MODEL ATTRIBUTE (package: com.ecommerce.frontendservice.config):
- Class: GlobalModelAttributes, annotated @ControllerAdvice
- A method annotated @ModelAttribute("loggedInUserName") that reads HttpSession attribute "userName" (inject HttpSession as a method parameter) and returns it, or null if not present — this makes the logged-in user's name available to every template automatically, including the header fragment, without each controller adding it manually

UPDATE: header fragment (src/main/resources/templates/fragments/header.html)
- If model attribute "loggedInUserName" is present (th:if="${loggedInUserName != null}"): show "Hello, [name]", a link to /orders (Order History), a link to /cart (Cart), and a logout form (a plain <form method="post" action="/logout"> with a submit button, since this is a state-changing action and should not be a GET link)
- If not present: show a link to /login and a link to /register

CONTROLLERS (package: com.ecommerce.frontendservice.controller):

AuthController
- GET /register — returns view "register" with an empty RegisterRequest in the model
- POST /register — @ModelAttribute RegisterRequest (plain form binding, not @Valid — validation happens server-side in user-service, we just surface whatever error it returns); call userServiceClient.register, catching feign.FeignException.Conflict AND feign.FeignException.BadRequest as described above; on success, add a flash attribute "message" = "Registration successful, please log in" and redirect to /login; on EmailAlreadyExistsException, add "error" = the exception's message to the Model and return to view "register"; on InvalidRegistrationException, add "error" = "Please check your details — name must contain only letters and spaces, and password must be at least 6 characters" to the Model and return to view "register" (same pattern as the email-conflict case — not a redirect, so form data + error show together)
- GET /login — returns view "login"
- POST /login — @ModelAttribute LoginRequest; call userServiceClient.login (catching feign.FeignException.Unauthorized as described above), then getCurrentUser, then populate the session as described above; on success redirect to "/"; on InvalidCredentialsException, add "error" to the Model and return to view "login"
- POST /logout — invalidates the HttpSession, redirects to "/"

CartController
- GET /cart — reads userId from session, calls orderServiceClient.getCart, returns view "cart" with the result as model attribute "cart"
- POST /cart/add — form fields productId and quantity; reads userId from session, calls orderServiceClient.addItem, redirects to "/cart"
- POST /cart/remove/{itemId} — reads userId from session, calls orderServiceClient.removeItem(userId, itemId), redirects to "/cart"

OrderController
- POST /checkout — reads userId from session, calls orderServiceClient.checkout (catching feign.FeignException.BadRequest as described above); on success returns view "order-confirmation" directly (not a redirect) with the result as model attribute "order", so the confirmation details are available in the same response; on EmptyCartException, add "error" as a flash attribute and redirect to "/cart"
- GET /orders — reads userId from session, calls orderServiceClient.getOrderHistory, returns view "orders" with the result as model attribute "orders"

UPDATE: product-detail.html
- Replace the existing placeholder comment with an add-to-cart form: <form method="post" action="/cart/add"> containing a hidden input for productId (value from the product's id), a number input for quantity (default value 1, min 1), and a submit button labeled "Add to Cart"
- If loggedInUserName is null, show a message "Please log in to add items to your cart" with a link to /login instead of the form

NEW THYMELEAF TEMPLATES (src/main/resources/templates/):

register.html
- A form with fields name, email, password, posting to /register
- Display the "error" model attribute if present
- Link to /login for existing users

login.html
- A form with fields email, password, posting to /login
- Display the "error" model attribute if present, and the "message" flash attribute if present (e.g. after a successful registration redirect)
- Link to /register for new users

cart.html
- Lists the "cart" model attribute's items: product name, quantity, price snapshot, a per-item remove form (small <form method="post" action="/cart/remove/{itemId}"> per row)
- Shows a computed total (sum of priceSnapshot × quantity across items) — compute this in the controller and pass it as a separate model attribute "cartTotal" rather than doing arithmetic in the template
- If the cart is empty, show "Your cart is empty" instead of a table
- A "Proceed to Checkout" form (<form method="post" action="/checkout">) shown only if the cart has at least one item

order-confirmation.html
- Displays the "order" model attribute: order id, status, items, and confirms the order was placed
- Link to /orders and link to home

orders.html
- Lists the "orders" model attribute: for each past order, show id, status, createdAt, and its items
- If empty, show "You have no past orders"

CONSTRAINTS:
- Do not use JavaScript anywhere, including for the logout action — it must be a real HTML form POST, not a link styled as a button.
- Do not write a custom Feign ErrorDecoder for either new client — catching the specific FeignException subtype at each call site (as described above) is sufficient and correct for this version.
- Do not add password confirmation fields, "remember me", or any auth feature beyond what's specified — keep the forms minimal.
- Do not modify UserServiceClient's or OrderServiceClient's error handling beyond the three specific exceptions listed — let any other FeignException type propagate up to the existing WebExceptionHandler from Part A.
- Do not add client-side validation of any kind (no HTML5 required/pattern attributes beyond what's trivial like input type="email") — validation is the backend services' job.
- Output every new/changed file in full: all new DTOs, both new Feign client interfaces, all three new exceptions, LoginRequiredInterceptor + its WebMvcConfigurer registration, GlobalModelAttributes, the updated header fragment, all three controllers, the updated product-detail.html, and all five new templates (register, login, cart, order-confirmation, orders).
- Do not attempt to deserialize or parse user-service's field-level validation error JSON on the frontend — InvalidRegistrationException carries a fixed, generic message only, as specified above. Field-level error mapping is out of scope for this version.

VERIFICATION (tell me how to confirm this works):
- List the exact browser click-path to test, in order, with eureka-server, product-service, user-service, order-service, and frontend-service all running: register a new account, get redirected to login with the success message visible, log in, confirm the header now shows your name, browse to a product and add it to cart, view the cart and confirm the total is correct, remove one item, add it back, check out, confirm the order confirmation page shows correct details, view order history and confirm the order appears, log out, and confirm /cart now redirects to /login.
```