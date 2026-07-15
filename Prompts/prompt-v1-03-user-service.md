# PROMPT 3 — user-service

> Paste everything in the code block below directly into Amazon Q as a single prompt.
> This service is self-contained — it does not call any other service.
> Assumes the project was generated via Spring Initializr per the settings above, PLUS the one manual pom.xml addition for spring-security-crypto described above.

```
I have already generated this Spring Boot project via Spring Initializr, with one manual addition to pom.xml — do not modify, add, or remove anything else in pom.xml, and do not change the Spring Boot or Spring Cloud version:

- Spring Boot 4.1.0, Spring Cloud release train 2025.1.2 (already configured in pom.xml)
- Group: com.ecommerce, Artifact: user-service, Package: com.ecommerce.userservice
- Dependencies present: Spring Web, Spring Data JPA, Validation, Eureka Discovery Client, PostgreSQL Driver, Lombok, and spring-security-crypto (added manually — NOT the full spring-boot-starter-security, which is intentionally absent since it would auto-configure a login form and lock every endpoint)
- Configuration format: application.yml (already exists, currently empty/default)

Your task is ONLY to write the application logic and configuration on top of this existing project. Do not touch pom.xml. Do not suggest adding spring-boot-starter-security or any other dependency beyond what's listed above.

MAIN APPLICATION CLASS:
- File: src/main/java/com/ecommerce/userservice/UserServiceApplication.java
- Annotations: @SpringBootApplication, @EnableDiscoveryClient

CONFIGURATION FILE:
- File: src/main/resources/application.yaml
- server.port: 8082
- spring.application.name: user-service
- spring.datasource.url: jdbc:postgresql://localhost:5432/project_db?currentSchema=user_schema
- spring.datasource.username: postgres
- spring.datasource.password: root
- spring.jpa.hibernate.ddl-auto: update
- spring.jpa.properties.hibernate.default_schema: user_schema
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

CONSTRAINTS:
- Do not suggest or add spring-boot-starter-security — this would activate a default login form and block all endpoints, which we do not want in v1. Only the standalone spring-security-crypto artifact (already present) should be used, purely for BCryptPasswordEncoder.
- Do not return the raw User entity from any controller method — always use UserResponse to avoid leaking passwordHash.
- Do not add JWT, token expiry, or any persistence for sessions — that is deferred to a later version.
- Do not add any endpoints beyond register, login, and /me.
- Output every file in full: application.yml, User entity, UserRepository, the BCryptPasswordEncoder config class, SessionStore, all four DTOs, UserController, all three custom exceptions, and GlobalExceptionHandler. Do NOT output pom.xml — it already exists and must not change.

VERIFICATION (tell me how to confirm this works):
- List the exact curl or Postman requests to test: register a user, attempt to register the same email again (expect 409), login with correct credentials (expect a token), login with wrong password (expect 401), call /me with the token in X-Session-Token header (expect the user's details, no passwordHash field present), call /me with no header (expect 401).
```