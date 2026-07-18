This is NOT a new project — you're continuing in the existing user-service project from v1. Do not run Spring Initializr again. I have already manually added three dependencies to pom.xml for this version (jjwt-api, jjwt-impl, jjwt-jackson, version 0.12.6) — do not modify, add to, or remove anything else in pom.xml, and do not suggest any further dependency.

Existing files in this project (do not regenerate these from scratch — you are extending/modifying them):
- User entity (com.ecommerce.userservice.entity.User) — unchanged, do not touch
- UserRepository — unchanged, do not touch
- BCryptPasswordEncoder config bean — unchanged, do not touch
- RegisterRequest, LoginRequest, UserResponse DTOs — unchanged, do not touch
- EmailAlreadyExistsException, InvalidCredentialsException, UnauthorizedException — unchanged, do not touch
- GlobalExceptionHandler — unchanged, do not touch, no new exception types are introduced in this version

CONFIGURATION FILE CHANGE:
- File: src/main/resources/application.yaml
- Add these three new keys (keep every existing key exactly as it is — datasource, JPA, Eureka, port, etc. — do not remove or modify anything already there):
  jwt.secret: <PASTE_YOUR_JWT_SECRET_HERE> — this exact string must also be placed in api-gateway's application.yml in a separate prompt; treat it as a fixed shared value, not something to regenerate
  jwt.expiration-ms: 7200000
  internal.secret: <PASTE_YOUR_INTERNAL_SECRET_HERE> — a SEPARATE, DIFFERENT string from jwt.secret above; this one will also be placed in api-gateway's, product-service's, and order-service's application.yml files in separate prompts; do not confuse the two values or use the same string for both

DELETE THIS FILE:
- src/main/java/com/ecommerce/userservice/session/SessionStore.java — the in-memory token map is no longer needed now that JWTs are self-contained. Delete this file entirely.

NEW CLASS (package: com.ecommerce.userservice.security):
- Class: JwtService, annotated @Component
- Reads jwt.secret and jwt.expiration-ms from application.yaml via @Value
- Method: String generateToken(User user) — builds a JWT using the jjwt 0.12.x fluent builder API (Jwts.builder()...), with these claims:
    - subject (sub): user.getId() converted to String
    - claim "email": user.getEmail()
    - claim "role": user.getRole()
    - issuedAt: now
    - expiration: now + the configured jwt.expiration-ms
    - signed with HMAC-SHA256 using a SecretKey derived from jwt.secret (use Keys.hmacShaKeyFor() on the UTF-8 bytes of the secret)
- Return the compact serialized JWT string (a plain String) — this is what LoginResponse.token will now contain, no change needed to the LoginResponse DTO shape itself

NEW CLASS (package: com.ecommerce.userservice.security):
- Class: InternalSecretFilter, extends OncePerRequestFilter, annotated @Component
- Reads internal.secret from application.yaml via @Value
- Register this filter at the highest precedence so it runs before anything else touches the request — use a separate @Configuration class (FilterConfig, package com.ecommerce.userservice.config) with a @Bean of type FilterRegistrationBean<InternalSecretFilter>, calling .setOrder(Ordered.HIGHEST_PRECEDENCE) explicitly, rather than relying on @Order on the filter class alone
- In doFilterInternal(request, response, filterChain):
    1. Read header "X-Internal-Secret" from the request
    2. If missing OR does not exactly equal the configured internal.secret value, write HTTP status 401 directly to the response, set content type application/json, write body {"message": "Forbidden — direct access not permitted"}, and RETURN without calling filterChain.doFilter() — the request must never reach any controller
    3. If it matches, call filterChain.doFilter(request, response) to let the request proceed normally
- This filter applies to EVERY endpoint in this service with NO exceptions — including /api/users/register and /api/users/login, which remain open to end-users only via api-gateway, never directly. There is no whitelist in this filter, unlike the separate JWT logic that will live in api-gateway.
- This filter has nothing to do with the JWT — do not attempt to parse or validate anything JWT-related here, it is a simple, unconditional string-equality check against one fixed header value.

MODIFY: UserController (package: com.ecommerce.userservice.controller)
- POST /api/users/register — NO CHANGE to this method's logic at all, leave it exactly as it is
- POST /api/users/login — change only the success path: instead of calling sessionStore.createSession(user.getId()), call jwtService.generateToken(user) and return that string in LoginResponse.token. The failure path (InvalidCredentialsException on bad email/password) is unchanged.
- GET /api/users/me — this method's logic changes completely:
    - Remove the X-Session-Token header parameter and the SessionStore lookup entirely
    - Instead, read two new headers: X-User-Id (String, parse to Long) and X-User-Email (String) — these are injected by api-gateway after it validates the JWT (built in a separate prompt); user-service does NOT re-validate the JWT itself, it trusts these headers (this trust is now reasonable specifically because InternalSecretFilter above already guarantees the request came through api-gateway, not a direct or spoofed call)
    - If X-User-Id is missing or blank, throw UnauthorizedException("Missing user identity — request must go through the gateway")
    - If present, parse the Long, look up the User by id via UserRepository.findById(); if not found throw UnauthorizedException("User not found")
    - Return 200 with UserResponse built from the looked-up User

UPDATE EXISTING TEST: UserControllerTest (package: com.ecommerce.userservice.controller, in src/test/java)
- Remove the @MockitoBean for SessionStore entirely — it no longer exists as a class
- Add a @MockitoBean for JwtService
- Test 4 (login_whenCredentialsAreCorrect_returns200WithToken) — change the mock from sessionStore.createSession(...) to jwtService.generateToken(any(User.class)), still returning a fixed mocked string; assertion stays the same
- Test 5 (getCurrentUser_whenTokenIsMissing_returns401) — change from "no X-Session-Token header" to "no X-User-Id header"; assertion stays 401
- Test 6 (getCurrentUser_whenTokenIsValid_returns200WithUserDetails) — change from mocking SessionStore.resolveUserId to instead sending the X-User-Id header directly in the MockMvc request and mocking UserRepository.findById() to return a matching User; assertion stays the same
- Tests 1, 2, 3 — unchanged, do not modify
- Do NOT add any test for InternalSecretFilter in this @WebMvcTest class — @WebMvcTest(UserController.class) does not load servlet filters registered via FilterRegistrationBean by default, so testing this filter here would be misleading; manual verification (below) is sufficient for this version, no automated test is required for the filter itself

CONSTRAINTS:
- Do not add a refresh-token endpoint or mechanism.
- Do not add role-based authorization checks anywhere.
- Do not modify User entity, UserRepository, RegisterRequest, LoginRequest, UserResponse, BCryptPasswordEncoder config, or any of the three existing custom exceptions.
- Do not touch pom.xml beyond what I've already manually added.
- Do not make InternalSecretFilter aware of or dependent on JwtService in any way — they are two completely independent mechanisms checking two completely different things.
- Output only the files that actually change: application.yaml (full file), the new JwtService class (full file), the new InternalSecretFilter class (full file), the new FilterConfig class (full file), the modified UserController (full file), and the modified UserControllerTest (full file). Do not regenerate or restate User, UserRepository, the DTOs, the exceptions, or GlobalExceptionHandler.

VERIFICATION (tell me how to confirm this works):
- First, confirm the filter works in isolation: with the service running standalone, send POST /api/users/login with valid credentials but NO X-Internal-Secret header — confirm 401 with the "Forbidden — direct access not permitted" message, and confirm this happens even though the credentials themselves would otherwise be valid (proving the filter runs before the controller). Repeat the same request WITH the correct X-Internal-Secret header — confirm it now proceeds and returns a token normally.
- Then confirm the JWT: decode the returned token manually at jwt.io using your jwt.secret value and confirm sub/email/role/exp claims are correct. Using the resolved user id, send GET /api/users/me with headers X-Internal-Secret (correct value) AND X-User-Id set to that id — confirm correct user details returned. Send the same request with the correct X-Internal-Secret but no X-User-Id — confirm 401 ("Missing user identity"). Send it with no X-Internal-Secret at all — confirm 401 ("Forbidden — direct access not permitted") regardless of what other headers are present.
- Also tell me the exact Maven command to run just UserControllerTest, and what a fully-passing console output should look like.