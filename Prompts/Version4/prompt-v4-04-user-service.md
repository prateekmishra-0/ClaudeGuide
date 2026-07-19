This is NOT a new project — you're continuing in the existing user-service project from v1/v2. Do not run Spring Initializr again. No dependency changes are needed for this version — do not modify pom.xml at all, do not suggest any addition.

Existing files in this project (do not regenerate these from scratch — you are extending/modifying them):
- OrderItem entity — unchanged, do not touch.
- Order entity — MODIFY, not unchanged: this entity's existing @PrePersist and @PreUpdate lifecycle hooks (built in v1) contain a hardcoded validation check that only permits status values "CART" and "PLACED" — anything else throws IllegalArgumentException. This version's checkout flow now sets "PENDING_PAYMENT", "PAID", and "PAYMENT_FAILED", none of which that check currently allows, which would cause every checkout to fail with a 500 the moment the order is saved. Update both hooks' validation to also accept these three new literal values, in addition to the existing "CART" and "PLACED" — do not remove or weaken the check itself, only widen the allowed set. No other change to the Order entity (fields, annotations, column definitions) — this is confined to the validation logic inside the two lifecycle methods.
- UserRepository — EXTENDED below (one new query method), everything else unchanged
- RegisterRequest, LoginRequest, LoginResponse, UserResponse DTOs — unchanged, do not touch
- SecurityConfig — unchanged, do not touch
- UserController — EXTENDED below (one new endpoint), no existing method changes
- EmailAlreadyExistsException, InvalidCredentialsException, UnauthorizedException — unchanged, do not touch
- GlobalExceptionHandler — EXTENDED below (one new exception mapping), no other changes
- JwtService, InternalSecretFilter, FilterConfig (from v2) — unchanged, do not touch

No configuration file change in this version — application.yml stays exactly as it is from v2.

MODIFY: UserRepository (package: com.ecommerce.userservice.repository)
- Add method: Optional<User> findById(Long id) — note: this almost certainly already exists implicitly, since UserRepository extends JpaRepository<User, Long>, which provides findById out of the box. Do not add it explicitly if it's already inherited — only add this line if, for some reason, this project's UserRepository does NOT extend JpaRepository directly. In the normal case, no change to this file is needed at all; state that explicitly rather than adding a redundant method.
- Do not add any other repository method.

NEW EXCEPTION (package: com.ecommerce.userservice.exception):
- Class: UserNotFoundException extends RuntimeException — thrown when a lookup by id finds no matching user. (This project's existing exceptions — EmailAlreadyExistsException, InvalidCredentialsException, UnauthorizedException — don't currently cover "no such user id," so this is a genuinely new exception type, not a duplicate of anything existing.)

MODIFY: GlobalExceptionHandler (package: com.ecommerce.userservice.exception, or wherever this project's existing handler package is — match its current location exactly)
- Add ONE new @ExceptionHandler method: UserNotFoundException → 404, with a clear message body following the exact same response shape this handler already uses for its other exception mappings (e.g. EmailAlreadyExistsException → 409). Do not change any existing mapping.

NEW DTO (package: com.ecommerce.userservice.dto):
- EmailResponse: email (String) — a single-field response DTO for the new endpoint below. Do not reuse UserResponse for this — UserResponse carries id/name/email/role and this endpoint should return only the one field it's actually for.

MODIFY: UserController (package: com.ecommerce.userservice.controller)
- Add new endpoint: GET /api/users/{id}/email
    - Calls userRepository.findById(id)
    - If not found, throw UserNotFoundException
    - If found, map to EmailResponse (just the email field) and return 200
- This is an internal-only endpoint by design — it exists solely for order-service to resolve a user's email at checkout time (see architecture-v4.md Section 6a). It is NOT meant to be reachable by end users through the gateway; that exposure gap is understood and is being closed separately in v5, not here. Do not add any ownership check, role check, or JWT-related logic to this endpoint in this prompt — that would be implementing a v5 feature ahead of schedule, which this project's own rules explicitly avoid.
- Do not modify any existing endpoint method in this controller (register, login, /me all stay exactly as they are).

CONSTRAINTS:
- Do not add authentication/authorization (ownership, role) checks anywhere on this new endpoint — explicitly out of scope for this prompt, deferred to v5.
- Do not add pagination — irrelevant, this returns a single record.
- Do not touch InternalSecretFilter or FilterConfig — this is a purely additive endpoint, the existing internal-secret mechanism from v2 already covers it automatically since the filter has no whitelist.
- Do not modify the User entity — this is a new read endpoint on existing data, not a schema change.
- Do not add unit tests in this prompt — manual verification via the steps below is sufficient for this version.
- Output only the files that actually change or are new: the new UserNotFoundException (full file), the new EmailResponse DTO (full file), the modified GlobalExceptionHandler (full file), and the modified UserController (full file). If UserRepository genuinely needs no change (the normal case, see note above), do not output it at all — just state explicitly that it already inherits findById and needs no edit. Do not regenerate or restate User entity, RegisterRequest, LoginRequest, LoginResponse, UserResponse, SecurityConfig, EmailAlreadyExistsException, InvalidCredentialsException, UnauthorizedException, JwtService, InternalSecretFilter, or FilterConfig.

VERIFICATION (tell me how to confirm this works):
- With user-service running standalone: send GET /api/users/{id}/email directly to port 8082 (with the correct X-Internal-Secret header) for a real, known user id — confirm it returns 200 with {"email": "..."} matching that user's actual registered email, and nothing else in the body.
- Send the same request for a user id that doesn't exist — confirm 404 via UserNotFoundException, with a response body matching this project's existing error shape (same as, say, a duplicate-email 409 already looks).
- Send the same request with NO X-Internal-Secret header — confirm 401, request never reaches the controller.
- Confirm all existing user-service endpoints (register, login, /me) still behave exactly as before — this change should be purely additive with zero regression.