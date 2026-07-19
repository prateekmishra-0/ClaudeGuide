This is NOT a new project — you're continuing in the existing frontend-service project from v1 (Parts A and B) and v2. Do not run Spring Initializr again. I have already manually added spring-boot-starter-validation to pom.xml for this version — do not modify, add to, or remove anything else in pom.xml, and do not suggest any further dependency.

Existing files in this project (do not regenerate these from scratch — you are extending/modifying them):
- UserServiceClient, OrderServiceClient Feign interfaces — unchanged, do not touch
- ProductServiceClient — EXTENDED below (one new method), existing methods unchanged
- All existing DTOs — unchanged, do not touch
- FeignAuthInterceptor — unchanged, do not touch; it applies automatically to any new Feign client call too
- HomeController, AuthController, CartController — unchanged, do not touch
- ProductController — MODIFIED below (one new fire-and-forget call), no other logic change
- OrderController — MODIFIED below (one new endpoint, one existing endpoint's signature changes), no other logic change
- LoginRequiredInterceptor, GlobalModelAttributes, WebExceptionHandler — unchanged, do not touch. LoginRequiredInterceptor already covers path pattern /checkout, so the new GET /checkout endpoint added below is automatically protected with zero configuration change.
- All existing exceptions — unchanged, do not touch
- home.html, product-detail.html, register.html, login.html, orders.html, error.html, header fragment — unchanged, do not touch
- cart.html — MODIFIED below (one small change to the checkout button)
- order-confirmation.html — MODIFIED below (status-aware messaging)

No configuration file change in this version — application.yaml stays exactly as it is from v2.

NEW DTOs (package: com.ecommerce.frontendservice.dto):

- ViewRequest: userId (Long) — request body for POST /api/products/{id}/view

- CheckoutForm: this is a form-backing object, validated locally in frontend-service since this data never reaches any backend service:
    - cardNumber: String, @NotBlank(message = "Card number is required"), @Pattern(regexp = "^\\d{16}$", message = "Card number must be exactly 16 digits")
    - cvv: String, @NotBlank(message = "CVV is required"), @Pattern(regexp = "^\\d{3}$", message = "CVV must be exactly 3 digits")
    - expiry: String, @NotBlank(message = "Expiry is required"), @Pattern(regexp = "^(0[1-9]|1[0-2])/\\d{2}$", message = "Expiry must be in MM/YY format")
    - Add one custom validation method annotated @AssertTrue(message = "Card has already expired"), e.g. isExpiryNotInPast() — parse the expiry string's MM/YY into a java.time.YearMonth (assume 20YY for the two-digit year) and return true if that YearMonth is equal to or after YearMonth.now(); if the string doesn't match the pattern at all, return true here and let the @Pattern annotation above report the format error instead (avoid double-reporting on the same field)

MODIFY: ProductServiceClient (package: com.ecommerce.frontendservice.client)
- Add: @PostMapping("/api/products/{id}/view") void recordView(@PathVariable("id") Long id, @RequestBody ViewRequest request)
- Purely declarative, no implementation body. Do not modify any existing method on this interface.

MODIFY: ProductController (package: com.ecommerce.frontendservice.controller)
- GET /products/{id} — after all existing logic (product lookup, and the recommendation-fetching addition from v3, both unchanged), add: read HttpSession attribute "userId" (inject HttpSession as a method parameter, same access pattern used elsewhere in this project). If present (user is logged in), call productServiceClient.recordView(id, new ViewRequest(userId)), wrapped in try/catch treating ANY exception as a no-op — this call must never block or break page rendering, and its result is never added to the Model; nothing in product-detail.html needs to change for this. If not logged in (attribute absent), skip this call entirely — do not call it with a null or placeholder userId.

MODIFY: OrderController (package: com.ecommerce.frontendservice.controller)

Add a NEW endpoint:
- GET /checkout — reads userId from session, calls orderServiceClient.getCart(userId). If the cart has zero items, redirect to "/cart" instead of showing an empty checkout page — do not call orderServiceClient.checkout in this branch, this is just a guard against someone navigating straight to /checkout via the URL bar with nothing in their cart. If the cart has items, compute cartTotal the same way CartController already does for cart.html (sum of priceSnapshot × quantity across items, computed here, not in the template), add "cart" and "cartTotal" to the Model, add a new empty CheckoutForm() to the Model under attribute name "checkoutForm", return view name "checkout".

MODIFY the existing POST /checkout endpoint:
- Change its signature to accept @Valid @ModelAttribute("checkoutForm") CheckoutForm form, BindingResult bindingResult, in addition to whatever it already reads from the session.
- FIRST check bindingResult.hasErrors(): if true, re-fetch the cart and recompute cartTotal exactly as GET /checkout does above, add both back to the Model along with the invalid checkoutForm (Spring MVC does this automatically via @ModelAttribute), and return view name "checkout" again — do NOT call orderServiceClient.checkout at all in this case. This re-displays the form with field-level errors and the cart still visible.
- If validation passes, proceed EXACTLY as this method's existing logic already does: call orderServiceClient.checkout(userId), with its existing feign.FeignException.BadRequest → EmptyCartException and feign.FeignException.Conflict → InsufficientStockException catches unchanged, on success return view "order-confirmation" with "order" as the model attribute, on either exception add "error" as a flash attribute and redirect to "/cart" — all of this exactly as it already works today.
- The CheckoutForm's fields (cardNumber, cvv, expiry) are read only for their validation side-effect. They must NEVER be passed to orderServiceClient.checkout(...), never added to any other Model attribute, never logged, and never stored in the session.

CONSTRAINTS:
- Do not use JavaScript anywhere on the new checkout page or anywhere else — same constraint as every other template in this project since v1. Card format enforcement is entirely server-side via CheckoutForm's Bean Validation annotations, not client-side.
- Do not add the Authorization header manually anywhere — FeignAuthInterceptor already attaches it automatically to every Feign call made by any client targeting "api-gateway", including the new recordView call.
- Do not add any internal-secret logic anywhere in this project — frontend-service never calls a backend service directly, that constraint from v2 is unchanged and applies here too.
- Do not add a new exception type for a view-tracking failure (it's caught and swallowed locally, fail-open, same principle as v3's recommendation calls) or for a card-validation failure (BindingResult handles that entirely within the controller — neither needs to reach WebExceptionHandler).
- Do not add caching anywhere.
- Do not modify HomeController, AuthController, CartController, LoginRequiredInterceptor, GlobalModelAttributes, WebExceptionHandler, or any existing exception.
- Output every file that changes or is new, in full: the modified ProductServiceClient, the new ViewRequest DTO, the new CheckoutForm DTO, the modified ProductController, the modified OrderController, the new checkout.html, the modified cart.html, and the modified order-confirmation.html. Do not regenerate or restate any other DTO, controller, template, interceptor, or exception.

NEW TEMPLATE: checkout.html (src/main/resources/templates/)
- Include the header fragment, same th:replace pattern used by every other page in this project
- Display a summary of the "cart" model attribute (same item-listing pattern already used in cart.html: product name, quantity, price snapshot per row) and the "cartTotal" model attribute
- Below the summary, a card-entry form: <form method="post" action="/checkout" th:object="${checkoutForm}">, with three text inputs bound via th:field — cardNumber (maxlength 16), cvv (maxlength 3), expiry (placeholder "MM/YY") — use type="text" for all three, not type="number", since the real validation is server-side and number inputs introduce unwanted browser-native spinner/leading-zero behavior
- Show field-level errors next to each field using th:if="${#fields.hasErrors('cardNumber')}" and th:errors="*{cardNumber}" (repeat for cvv, expiry)
- A submit button labeled "Pay Now"

MODIFY: cart.html
- The existing "Proceed to Checkout" form (currently <form method="post" action="/checkout">) changes to a plain GET link instead: <a href="/checkout">Proceed to Checkout</a> — still shown only if the cart has at least one item, same condition as before. This is because checkout is now a two-step flow (view the card form first, then submit), not a single POST straight from the cart page. No other part of cart.html changes.

MODIFY: order-confirmation.html
- Currently shows a generic "order placed" message regardless of outcome. Branch on the "order" model attribute's status field using th:if/th:unless (or th:switch): if status equals "PAID", show a clear success message (e.g. "Payment successful — your order has been placed."); if status equals "PAYMENT_FAILED", show a clear failure message instead (e.g. "Payment failed. Your items have been returned to stock — please try again."). Still display the order id and items in both cases, only the headline message (and optionally a plain-CSS visual cue, no JS) differs.

VERIFICATION (tell me how to confirm this works):
- With eureka-server, api-gateway, product-service, user-service, order-service, payment-service, and notification-service all running: log in, visit a product detail page, then check product-service's GET /api/products/{id}/views/count-by-viewer directly — confirm a view was recorded for your user id. Visit the same product a second time — confirm a second row was recorded (not an incremented count), matching the no-uniqueness-constraint design. Log out and visit a product page — confirm no view is recorded for an anonymous visit.
- Add an item to your cart, click "Proceed to Checkout" — confirm you land on the new checkout page showing your cart summary and total, with a blank card form, NOT an immediate order confirmation.
- Submit the form with an invalid card number (e.g. 15 digits) — confirm the page re-displays with a field-level error next to that field, the cart summary is still visible, and no new order was actually created in order-service.
- Submit with an expired date (e.g. 01/20) — confirm the "Card has already expired" error appears and checkout is blocked.
- Submit with a valid 16-digit card, 3-digit CVV, and a future MM/YY — confirm checkout proceeds and you land on order-confirmation.html.
- Confirm — by checking order-service's actual request logs or database — that the card number, CVV, and expiry you entered never appear anywhere outside frontend-service.
- Complete one checkout with a total that will succeed (PAID) — confirm order-confirmation.html shows the success message. Complete a second checkout with a total ending in .13 cents (PAYMENT_FAILED) — confirm the page shows the failure message instead, while still listing the order's items correctly.
- Manually navigate to /checkout via the URL bar with an empty cart — confirm you're redirected to /cart, not shown a broken empty checkout form.