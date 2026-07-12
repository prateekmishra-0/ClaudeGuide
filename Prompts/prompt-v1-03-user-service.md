# PROMPT 3 — user-service

> Paste everything in the code block below directly into Amazon Q as a single prompt.
> This service is self-contained — it does not call any other service.

```
Create a Spring Boot microservice from scratch called user-service.

PROJECT SETUP:
- Build tool: Maven
- Java version: 17
- Spring Boot version: 3.2.x
- Group: com.ecommerce
- Artifact: user-service
- Package: com.ecommerce.userservice
- Packaging: Jar

DEPENDENCIES (pom.xml):
- spring-boot-starter-web
- spring-boot-starter-data-jpa
- spring-boot-starter-validation
- spring-cloud-starter-netflix-eureka-client
- Spring Cloud version: 2023.0.x (compatible with Spring Boot 3.2.x) — add spring-cloud-dependencies BOM in dependencyManagement
- postgresql (runtime scope)
- spring-security-crypto — add ONLY this artifact (org.springframework.security:spring-security-crypto), NOT spring-boot-starter-security. We need BCryptPasswordEncoder only, not a full security filter chain or auto-configured login form.
- lombok (optional scope)

MAIN APPLICATION CLASS:
- Class name: UserServiceApplication
- Location: com.ecommerce.userservice
- Annotations: @SpringBootApplication, @EnableDiscoveryClient

CONFIGURATION FILE:
- File: src/main/resources/application.yml
- server.port: 8082
- spring.application.name: user-service
- spring.datasource.url: jdbc:postgresql://localhost:5432/ecommerce_db?currentSchema=user_schema
- spring.datasource.username: postgres
- spring.datasource.password: postgres
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
- A @Configuration class exposing a @Bean of type BCryptPasswordEncoder (from spring-security-crypto)

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
- Do not add Spring Security's full auto-configuration (no spring-boot-starter-security dependency) — this would activate a default login form and block all endpoints, which we do not want in v1.
- Do not return the raw User entity from any controller method — always use UserResponse to avoid leaking passwordHash.
- Do not add JWT, token expiry, or any persistence for sessions — that is deferred to a later version.
- Do not add any endpoints beyond register, login, and /me.
- Output every file in full: pom.xml, application.yml, User entity, UserRepository, the BCryptPasswordEncoder config class, SessionStore, all four DTOs, UserController, all three custom exceptions, and GlobalExceptionHandler.

VERIFICATION (tell me how to confirm this works):
- List the exact curl or Postman requests to test: register a user, attempt to register the same email again (expect 409), login with correct credentials (expect a token), login with wrong password (expect 401), call /me with the token in X-Session-Token header (expect the user's details, no passwordHash field present), call /me with no header (expect 401).
```
