This is NOT a new project — you're continuing in the existing order-service project from v1/v2/v3. Do not run Spring Initializr again. No dependency changes are needed for this version — do not modify pom.xml at all, do not suggest any addition.

Existing files in this project (do not regenerate these from scratch — you are extending/modifying them):
- Order, OrderItem entities — unchanged, do not touch. Order.status stays a plain String column (@Column(length = 30), established in v1) — no schema change, this version just introduces new valid literal values used in code.
- OrderRepository, OrderItemRepository — unchanged, do not touch (the v3 addition, findByProductIdAndOrder_Status, stays as-is)
- AddItemRequest, OrderItemResponse, OrderResponse, ProductResponse, StockAdjustRequest DTOs — unchanged, do not touch
- ProductServiceClient Feign interface — unchanged, do not touch. Its adjustStock method is already correctly annotated @PutMapping("/api/products/{id}/stock") — confirmed against the real file — and is reused as-is for both the existing stock decrement (step 2 below) and this version's new restock-on-failure call (step 7b below). Do not add a second Feign client method for this.
- OrderController — unchanged, do not touch. The checkout endpoint's signature, request handling, and response mapping to OrderResponse all stay exactly as they are — only the underlying OrderService logic changes in this prompt, not the controller.
- ProductNotFoundException, OrderItemNotFoundException, EmptyCartException, InsufficientStockException — unchanged, do not touch. No new exception type is introduced in this version — the payment-service call is deliberately left to propagate any failure as-is (see Constraints).
- GlobalExceptionHandler — unchanged, do not touch.
- InternalSecretFilter, FilterConfig (from v2) — unchanged, do not touch.

No configuration file change in this version — application.yaml stays exactly as it is from v3.

NEW DTOs (package: com.ecommerce.orderservice.dto):
- PaymentRequest: orderId (Long), amount (BigDecimal) — request body sent to payment-service. Note: this DTO must NEVER include any card-related field (card number, CVV, expiry) — the frontend's card-entry form (architecture-v4.md Section 8.2) is a cosmetic step contained entirely within frontend-service and its data never reaches order-service or payment-service at all. Do not add such fields here under any circumstance.
- PaymentResponse: status (String), paymentId (Long) — used to deserialize payment-service's response. payment-service actually returns more fields (orderId, amount, createdAt) — Jackson will ignore anything not declared here, so only declare what this service actually reads.
- EmailResponse: email (String) — used to deserialize user-service's GET /api/users/{id}/email response.
- NotificationRequest: orderId (Long), email (String), message (String) — request body sent to notification-service.

NEW FEIGN CLIENTS (package: com.ecommerce.orderservice.client):

Interface: PaymentServiceClient, @FeignClient(name = "payment-service")
- @PostMapping("/api/payments") PaymentResponse processPayment(@RequestBody PaymentRequest request)
- Purely declarative, no implementation body.

Interface: UserServiceClient, @FeignClient(name = "user-service")
- @GetMapping("/api/users/{id}/email") EmailResponse getUserEmail(@PathVariable("id") Long id)
- Purely declarative, no implementation body.

Interface: NotificationServiceClient, @FeignClient(name = "notification-service")
- @PostMapping("/api/notifications") void sendNotification(@RequestBody NotificationRequest request)
- Purely declarative, no implementation body.

All three resolve via Eureka directly, same pattern as the existing ProductServiceClient — none of this traffic goes through api-gateway. The existing FeignInternalSecretInterceptor (v2) is a globally-picked-up RequestInterceptor bean, so it automatically attaches X-Internal-Secret to all three of these new clients too — no per-client wiring needed, do not touch FeignInternalSecretInterceptor. Add all three as new `final` fields on OrderService, constructor-injected via the existing @RequiredArgsConstructor annotation, exactly the same way productServiceClient is already declared — do not add any manual constructor code or @Autowired field injection.

MODIFY: OrderService (package: com.ecommerce.orderservice.service)

Rewrite the existing checkout(Long userId) method (do NOT touch getOrCreateCart, addItemToCart, removeItemFromCart, or getOrderHistory — their logic is completely unaffected by this prompt):

1. (unchanged from v2/v3) Load the CART order and its items. If the cart is empty, throw EmptyCartException, same as before. For every item, call product-service GET /api/products/{id}, confirm stockQuantity >= requested quantity — check ALL items before mutating ANYTHING, same as v2. If any item fails, abort with 409 via InsufficientStockException, cart untouched.
2. If all items pass: for EVERY item, call product-service PUT /api/products/{id}/stock with delta = -quantity, via the existing productServiceClient.adjustStock(...) method (unchanged from v2 — this call itself is not being modified, just described accurately here).
3. Set the order's status to "PENDING_PAYMENT" using a raw String literal, exactly the same style this class already uses for "CART" and "PLACED" (e.g. cart.setStatus("PLACED") in the current checkout method, orderRepository.findByUserIdAndStatus(userId, "CART") elsewhere in this class) — do NOT introduce private static final String constants for these values. This class has no such constants for its existing two status values, and introducing them now for only the three new ones would be an inconsistent, unrequested style change within the same method. Use the literal strings "PENDING_PAYMENT", "PAID", and "PAYMENT_FAILED" directly wherever a status needs to be set below, matching the existing "CART"/"PLACED" convention exactly.
4. Compute the order total: sum of (item.getPriceSnapshot() times item.getQuantity()) across all items, using BigDecimal arithmetic (multiply, then add/reduce — do not use double/float anywhere for this).
5. Call paymentServiceClient.processPayment(new PaymentRequest(order.getId(), total)) — DO NOT wrap this specific call in try/catch. If payment-service is unreachable or throws, let the exception propagate normally up through the existing call stack (it will be caught by GlobalExceptionHandler's generic 500 fallback, same as any other unhandled exception in this service today). This is a deliberate, stated gap — resilience for this specific call is v5's job (circuit breaker), not this version's.
6. Attempt to resolve the user's email, wrapped in try/catch treating ANY exception (network failure, 404, anything) as "no email available" — call userServiceClient.getUserEmail(userId), store the result if it succeeds, proceed with a null/empty email if it fails. This failure must never abort or alter the checkout's outcome in any way.
7. Branch on the payment response's status field:
   a. If "SUCCESS": set order status to the literal "PAID". If an email was resolved in step 6, call notificationServiceClient.sendNotification(...) with a success message (e.g. "Your order #" + orderId + " has been placed and paid successfully."), wrapped in try/catch treating any failure as a no-op — never let a notification failure affect the checkout's response.
   b. If "FAILED": set order status to the literal "PAYMENT_FAILED". For EVERY item in the order, call product-service PUT /api/products/{id}/stock with a POSITIVE delta equal to that item's quantity (restocking what step 2 decremented) — reuse the same productServiceClient.adjustStock(...) method as step 2, just with the sign flipped; this call is NOT wrapped in try/catch, it should behave with the same lack-of-resilience as every other product-service stock call in this service today. If an email was resolved in step 6, call notificationServiceClient.sendNotification(...) with a failure message (e.g. "Your order #" + orderId + " could not be completed — payment failed and your items have been restocked."), wrapped in try/catch treating any failure as a no-op, same as 7a.
8. Save the order with its final status and return it (mapped to OrderResponse by the controller, unchanged).

CONSTRAINTS:
- Do not add retry, circuit breaker, or timeout logic on ANY of the three new outbound calls (payment, user-email, notification) — that's v5's job entirely, for all three, not just the payment call.
- Do not add caching anywhere.
- Do not add a new exception type for a payment-service failure — an unreachable/erroring payment-service is allowed to surface as the existing generic 500 behavior, this is intentional for this version.
- Do not add authentication/authorization (userId ownership) checks anywhere — same stated known limitation as the rest of order-service currently has, not addressed here.
- Do not modify Order or OrderItem entities — no schema change, this is new logic on existing columns.
- Do not touch InternalSecretFilter, FilterConfig, or FeignInternalSecretInterceptor — the existing mechanism already covers all three new outbound calls automatically.
- Do not introduce any private static final String status constants anywhere in this class — use raw String literals throughout, matching the existing "CART"/"PLACED" pattern exactly, as described in step 3 above.

UNIT TEST CHANGES (package: com.ecommerce.orderservice.service, in src/test/java, class OrderServiceTest):

This version genuinely requires touching existing tests, unlike the purely-additive v3 order-service prompt — checkout's actual behavior has changed (it no longer produces a PLACED-status order under any circumstance). This class currently has 10 tests. Apply exactly these changes, no others:

- Test 4 (checkout_whenCartHasItems_setsStatusToPlacedAndSaves) — MUST be rewritten. It currently asserts the resulting order's status equals "PLACED", which is no longer possible from any checkout path in the rewritten method. Repurpose or remove this test so nothing in the final suite asserts a final status of "PLACED" — its intent is effectively superseded by the new checkout_whenPaymentSucceeds_setsStatusPaidAndDoesNotRestock test below, so folding its setup into that new test and removing this one as a separate test is acceptable.

- Test 7 (checkout_whenAllItemsHaveSufficientStock_callsAdjustStockForEveryItemWithCorrectNegativeDelta) — needs ONE addition, not a rewrite: this test's flow now reaches the new payment call partway through, since it exercises the full checkout path past the stock-validation and stock-decrement steps. Add a mock stub for paymentServiceClient.processPayment(...) returning a SUCCESS PaymentResponse, so the test doesn't NPE on an unstubbed call before its existing assertion (verifying adjustStock was called with the correct negative delta for every item) is ever reached. The existing assertion itself does not need to change.

- Tests 3, 6, 8 (checkout_whenCartHasZeroItems_throwsEmptyCartException, checkout_whenAnyItemHasInsufficientStock_throwsInsufficientStockException_andNeverCallsAdjustStock, checkout_whenAdjustStockThrowsConflictDuringMutationPass_throwsInsufficientStockException) — NO CHANGES. All three throw before the rewritten method ever reaches the new payment call, so their existing mocks and assertions remain valid exactly as they are.

- Tests 1, 2, 5, 9, 10 — NO CHANGES. None of these touch checkout(...) at all (they cover addItemToCart, getOrCreateCart, and v3's getOrdersContainingProduct) and are entirely unaffected by this version's changes.

In addition, ADD these new tests:
- checkout_whenPaymentSucceeds_setsStatusPaidAndDoesNotRestock — mock a SUCCESS payment response, verify final order status equals the literal "PAID", verify product-service's adjustStock method is NOT called again after the initial decrement (no restock)
- checkout_whenPaymentFails_setsStatusPaymentFailedAndRestocksAllItems — mock a FAILED payment response, verify final order status equals the literal "PAYMENT_FAILED", verify product-service's adjustStock method IS called again for every item with a positive delta matching that item's quantity
- checkout_whenEmailLookupFails_stillCompletesCheckoutSuccessfully — mock the user-service client to throw an exception, verify checkout still completes and returns the correct final status (either "PAID" or "PAYMENT_FAILED" depending on the payment mock in that test), and verify notificationServiceClient.sendNotification is never called in this case
- checkout_whenNotificationCallFails_stillReturnsCorrectOrderStatus — mock a successful payment AND mock the notification client to throw, verify checkout still completes and returns "PAID", proving a notification failure never surfaces as a checkout error

CONSTRAINTS on tests:
- Do not remove or weaken the existing stock-validation tests (insufficient stock aborting checkout, cart-empty aborting checkout) — those parts of the flow are unchanged from v2/v3 and their tests should still pass as-is.
- Do not touch, rename, or renumber Tests 1, 2, 3, 5, 6, 8, 9, or 10 beyond what's specified above.
- Output the full modified OrderServiceTest file, including all pre-existing tests (updated where needed) plus the four new ones described above — do not truncate or summarize any existing test.

CONSTRAINTS on output:
- Output only the files that actually change or are new: the three new Feign client interfaces (full files), the four new DTOs (full files), the modified OrderService (full file), and the modified OrderServiceTest (full file). Do not regenerate or restate Order, OrderItem, OrderRepository, OrderItemRepository, AddItemRequest, OrderItemResponse, OrderResponse, ProductResponse, StockAdjustRequest, ProductServiceClient, OrderController, any exception, GlobalExceptionHandler, InternalSecretFilter, FilterConfig, or FeignInternalSecretInterceptor.

VERIFICATION (tell me how to confirm this works):
- Prerequisite: eureka-server, product-service, user-service, payment-service, and notification-service must all be running and registered. Have a test user with a real inbox-checkable email registered.
- Add items to a cart and checkout normally with a total that will NOT trigger payment-service's .13-cents rule — confirm the response shows status PAID, confirm product-service's stock was decremented and NOT restocked, and confirm a real success email arrives in the test user's inbox.
- Add items to a cart with a total that WILL trigger the .13-cents rule (e.g. a single item priced so the total ends in .13) and checkout — confirm status PAYMENT_FAILED, confirm product-service's stock was decremented then correctly restocked back to its original value, and confirm a real failure email arrives.
- Temporarily stop user-service, then perform a normal successful checkout — confirm checkout still completes and returns PAID correctly, confirm no notification email is sent (since the email couldn't be resolved), and confirm no error is returned to the caller. Restart user-service afterward.
- Temporarily stop notification-service, then perform a normal successful checkout — confirm checkout still completes and returns PAID correctly despite the notification call failing. Restart notification-service afterward.
- Temporarily stop payment-service entirely, then attempt a checkout — confirm this DOES surface as an error (500) back to the caller, proving the payment call is intentionally left unguarded in this version, unlike the email/notification calls.
- Tell me the exact Maven command to run just OrderServiceTest, confirm all tests pass (old ones updated correctly, four new ones included), and describe what a fully-passing console output should look like.