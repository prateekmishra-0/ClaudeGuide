# PROMPT 3 (FINAL, CORRECTED) — user-service

> Paste everything in the code block below directly into Amazon Q as a single prompt.
> This service is self-contained — it does not call any other service.
> Assumes the project was already generated via Spring Initializr per the settings given earlier (Spring Boot 4.1.0, Group com.ecommerce, Artifact user-service, Package com.ecommerce.userservice, dependencies: Spring Web, Spring Data JPA, Validation, Eureka Discovery Client, PostgreSQL Driver, Lombok, PLUS the manual pom.xml addition of spring-security-crypto).
>
> **Correction from the original v1 prompt:** Spring Boot 4.0 removed `@MockBean`/`@SpyBean` entirely (deprecated since 3.4, gone since 4.0). Since this project targets 4.1.0, the test class below uses `@MockitoBean` (package `org.springframework.test.context.bean.override.mockito.MockitoBean`) instead. This is a straight annotation swap — `@MockitoBean` comes from `spring-test`, which `spring-boot-starter-test` already pulls in transitively, so no pom.xml change is needed.

```
I have already generated this Spring Boot project via Spring Initializr, with one manual addition to pom.xml — do not modify, add, or remove anything else in pom.xml, and do not change the Spring Boot or Spring Cloud version:

- Spring Boot 4.1.0, Spring Cloud release train 2025.1.2 (already configured in pom.xml)
- Group: com.ecommerce, Artifact: user-service, Package: com.ecommerce.userservice
- Dependencies present: Spring Web, Spring Data JPA, Validation, Eureka Discovery Client, PostgreSQL Driver, Lombok, and spring-security-crypto (added manually — NOT the full spring-boot-starter-security, which is intentionally absent since it would auto-configure a login form and lock every endpoint). spring-boot-starter-test is included by default, no action needed for it.
- Configuration format: application.yml (already exists, currently empty/default)

Your task is ONLY to write the application logic, configuration, and tests on top of this existing project. Do not touch pom.xml. Do not suggest adding spring-boot-starter-security or any other dependency beyond what's listed above.

MAIN APPLICATION CLASS:
- File: src/main/java/com/ecommerce/userservice/UserServiceApplication.java
- Annotations: @SpringBootApplication, @EnableDiscoveryClient

CONFIGURATION FILE:
- File: src/main/resources/application.yaml
- server.port: 8082
- spring.application.name: user-service
- spring.datasource.url: jdbc:postgresql://localhost:5432/user_db
- spring.datasource.username: postgres
- spring.datasource.password: root
- spring.jpa.hibernate.ddl-auto: update
- spring.jpa.show-sql: true
- eureka.client.service-url.defaultZone: http://localhost:8761/eureka

ENTITY (package: com.ecommerce.userservice.entity):

Entity: User
- id: Long, @Id, @GeneratedValue(strategy = GenerationType.IDENTITY)
- name: String, @NotBlank, @Column(length = 60)
- email: String, @NotBlank, @Email, @Column(unique = true)
- passwordHash: String, @NotBlank — stores the BCrypt hash only, never plaintext
- role: String, @Column(length = 20) — default value "CUSTOMER", set inside a @PrePersist method if not already set; do not expose a way for the client to set this via the register endpoint
- createdAt: LocalDateTime — set only inside a @PrePersist method, @Column(updatable = false)

Use Lombok @Getter @Setter @NoArgsConstructor @AllArgsConstructor.

REPOSITORY (package: com.ecommerce.userservice.repository):
- UserRepository extends JpaRepository<User, Long>
  - Add method: Optional<User> findByEmail(String email)
  - Add method: boolean existsByEmail(String email)

BEAN CONFIGURATION (package: com.ecommerce.userservice.config):
- A @Configuration class exposing a @Bean of type BCryptPasswordEncoder (from spring-security-crypto, already present in pom.xml)

SESSION STORE (package: com.ecommerce.userservice.session):
- A @Component class SessionStore holding an in-memory ConcurrentHashMap<String, Long> (token -> userId)
- Method: String createSession(Long userId) — generates a token via UUID.randomUUID().toString(), stores the mapping, returns the token
- Method: Optional<Long> resolveUserId(String token) — looks up the token, returns empty if not found
- This is intentionally NOT persisted to the database — state is lost on service restart, which is a known and accepted v1 limitation, do not add any persistence for this.

DTOs (package: com.ecommerce.userservice.dto):
- RegisterRequest: name (String, @NotBlank), email (String, @NotBlank @Email), password (String, @NotBlank, plaintext, minimum length 6 via @Size)
- LoginRequest: email (String, @NotBlank), password (String, @NotBlank)
- LoginResponse: token (String)
- UserResponse: id (Long), name (String), email (String), role (String) — used for the /me endpoint response, must NOT include passwordHash

CONTROLLER (package: com.ecommerce.userservice.controller):

UserController — base path /api/users
- POST /api/users/register — @Valid @RequestBody RegisterRequest; if existsByEmail is true, throw EmailAlreadyExistsException (→ 409); otherwise hash the password with BCryptPasswordEncoder, save the User, return 201 with UserResponse (not the raw entity, so passwordHash is never serialized)
- POST /api/users/login — @Valid @RequestBody LoginRequest; find user by email; if not found OR password doesn't match via BCryptPasswordEncoder.matches(), throw InvalidCredentialsException (→ 401); otherwise call SessionStore.createSession and return 200 with LoginResponse
- GET /api/users/me — reads a header named X-Session-Token; if missing or SessionStore.resolveUserId returns empty, throw UnauthorizedException (→ 401); otherwise load the user by the resolved id and return 200 with UserResponse

EXCEPTION HANDLING (package: com.ecommerce.userservice.exception):
- Custom exception: EmailAlreadyExistsException extends RuntimeException (constructor takes a String message)
- Custom exception: InvalidCredentialsException extends RuntimeException (constructor takes a String message)
- Custom exception: UnauthorizedException extends RuntimeException (constructor takes a String message)
- Class: GlobalExceptionHandler, annotated @RestControllerAdvice
  - @ExceptionHandler(MethodArgumentNotValidException.class) → 400, return a body listing each invalid field and its validation message
  - @ExceptionHandler(EmailAlreadyExistsException.class) → 409, return { "message": <exception message> }
  - @ExceptionHandler(InvalidCredentialsException.class) → 401, return { "message": <exception message> }
  - @ExceptionHandler(UnauthorizedException.class) → 401, return { "message": <exception message> }
  - @ExceptionHandler(Exception.class) → 500, generic fallback, return { "message": "An unexpected error occurred" }

UNIT TESTS (package: com.ecommerce.userservice.controller, in src/test/java):

IMPORTANT — Spring Boot version note: this project targets Spring Boot 4.1.0, where `@MockBean`/`@SpyBean` have been removed entirely (they were deprecated in 3.4 and removed in 4.0). Use `@MockitoBean` instead, imported from `org.springframework.test.context.bean.override.mockito.MockitoBean` (part of `spring-test`, already pulled in transitively by `spring-boot-starter-test` — no pom.xml change). Do NOT use `@MockBean` or its package `org.springframework.boot.test.mock.mockito.MockBean` anywhere — it will not compile on this version.

Test class: UserControllerTest
- Annotated @WebMvcTest(UserController.class) — loads only the web layer, not the full application context or a real database
- Mock UserRepository with @MockitoBean (NOT @MockBean)
- Mock SessionStore with @MockitoBean (NOT @MockBean)
- Do NOT mock BCryptPasswordEncoder — use the real bean (it's cheap, pure computation, and testing the actual hash/match logic is the point); if @WebMvcTest doesn't pick up the @Configuration class automatically, provide the real BCryptPasswordEncoder bean via a @TestConfiguration inner class in the test
1. register_whenEmailAlreadyExists_returns409 — mock existsByEmail to return true, POST /api/users/register with any valid body, assert status 409, assert the response body's "message" is present
2. register_whenNewEmail_returns201AndResponseExcludesPasswordHash — mock existsByEmail to return false and save() to return a User with an assigned id, POST a valid body, assert status 201, assert the response JSON does NOT contain a "passwordHash" key at all (not just that it's null — the field must be genuinely absent)
3. login_whenPasswordDoesNotMatch_returns401 — mock findByEmail to return a User whose passwordHash is a real BCrypt hash of some known password, POST /api/users/login with a different password, assert status 401
4. login_whenCredentialsAreCorrect_returns200WithToken — mock findByEmail to return a User whose passwordHash is a real BCrypt hash of a known password, mock SessionStore.createSession to return a fixed token string, POST with the matching plaintext password, assert status 200 and that the response body's "token" field equals the mocked token
5. getCurrentUser_whenTokenIsMissing_returns401 — send GET /api/users/me with no X-Session-Token header, assert status 401
6. getCurrentUser_whenTokenIsValid_returns200WithUserDetails — mock SessionStore.resolveUserId to return Optional.of(some id), mock UserRepository to return a matching User, send GET /api/users/me with a header, assert status 200 and correct name/email/role in the response

CONSTRAINTS:
- Do not suggest or add spring-boot-starter-security — this would activate a default login form and block all endpoints, which we do not want in v1. Only the standalone spring-security-crypto artifact (already present) should be used, purely for BCryptPasswordEncoder.
- Do not return the raw User entity from any controller method — always use UserResponse to avoid leaking passwordHash.
- Do not add JWT, token expiry, or any persistence for sessions — that is deferred to a later version.
- Do not add any endpoints beyond register, login, and /me.
- Do not add @SpringBootTest, Testcontainers, or a real Postgres connection anywhere in the tests.
- Do not use @MockBean anywhere — use @MockitoBean as specified above; this is a Spring Boot 4.1.0 project and @MockBean no longer exists in this version.
- Output every file in full: application.yml, User entity, UserRepository, the BCryptPasswordEncoder config class, SessionStore, all four DTOs, UserController, all three custom exceptions, GlobalExceptionHandler, and UserControllerTest. Do NOT output pom.xml — it already exists and must not change.

IMPORTANT — Spring Boot 4.1.0 import paths: use exactly these two imports in the test class, nothing else:
import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
Do NOT use org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest or org.springframework.boot.test.mock.mockito.MockBean — both are Spring Boot 3.x-era paths that no longer exist in 4.1.0.

VERIFICATION (tell me how to confirm this works):
- List the exact curl or Postman requests to test: register a user, attempt to register the same email again (expect 409), login with correct credentials (expect a token), login with wrong password (expect 401), call /me with the token in X-Session-Token header (expect the user's details, no passwordHash field present), call /me with no header (expect 401).
- Also tell me the exact Maven command to run just UserControllerTest, and what a fully-passing console output should look like.
```