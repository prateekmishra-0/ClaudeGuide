I have already generated this Spring Boot project via Spring Initializr — do not modify, add, or remove anything in pom.xml, and do not change the Spring Boot or Spring Cloud version:

- Spring Boot 4.1.0, Spring Cloud release train 2025.1.2 (already configured in pom.xml)
- Group: com.ecommerce, Artifact: notification-service, Package: com.ecommerce.notificationservice
- Dependencies present: Spring Web, Eureka Discovery Client, Lombok, Spring Boot Starter Mail (spring-boot-starter-test is included by default, no action needed for it)
- Configuration format: application.yml (already exists, currently empty/default)
- This service has NO database — no Postgres driver, no Spring Data JPA. It is stateless.
- This service does NOT call any other service — no Feign client, no @EnableFeignClients needed.

Your task is ONLY to write the application logic and configuration on top of this existing project. Do not touch pom.xml. Do not suggest adding any dependency beyond what's listed above.

MAIN APPLICATION CLASS:
- File: src/main/java/com/ecommerce/notificationservice/NotificationServiceApplication.java
- Annotations: @SpringBootApplication, @EnableDiscoveryClient
- Do NOT add @EnableFeignClients — this service never calls another service.

CONFIGURATION FILE:
- File: src/main/resources/application.yaml
- server.port: 8086
- spring.application.name: notification-service
- eureka.client.service-url.defaultZone: http://localhost:8761/eureka
- internal.secret: arhjYkqjrB6INMZvZjOApL6jk0b6LWJVnsEYl29kMY0= — this MUST be the exact same value already present in api-gateway's, product-service's, user-service's, order-service's, and payment-service's application.yml/application.yaml files. Do not generate a new value.
- spring.mail configuration (fill in the placeholder values with what I give you separately, do not invent test values):
  spring:
  mail:
  host: smtp.gmail.com
  port: 587
  username: YOUR_GMAIL_ADDRESS_HERE
  password: YOUR_16_CHARACTER_APP_PASSWORD_HERE
  properties:
  mail:
  smtp:
  auth: true
  starttls:
  enable: true

DTO (package: com.ecommerce.notificationservice.dto):
- NotificationRequest: orderId (Long, @NotNull), email (String, @NotBlank, @Email), message (String, @NotBlank) — request body for POST /api/notifications

SECURITY — INCOMING (package: com.ecommerce.notificationservice.security):
- Class: InternalSecretFilter, extends OncePerRequestFilter, @Component
- Reads internal.secret via @Value
- In doFilterInternal: read header "X-Internal-Secret"; if missing or doesn't exactly match the configured value, write HTTP 401 directly, content type application/json, body {"message": "Forbidden — direct access not permitted"}, return without calling filterChain.doFilter(); otherwise call filterChain.doFilter()
- Applies to EVERY endpoint in this service, no whitelist — including POST /api/notifications

NEW CLASS (package: com.ecommerce.notificationservice.config):
- Class: FilterConfig, @Configuration
- A @Bean method of type FilterRegistrationBean<InternalSecretFilter>, .setOrder(Ordered.HIGHEST_PRECEDENCE)

SERVICE LAYER (package: com.ecommerce.notificationservice.service):
- Class: NotificationService, @Service
- Inject JavaMailSender (auto-configured by Spring Boot once spring.mail.* properties are set — no manual bean needed)

Method: void sendNotification(NotificationRequest request)
1. Build a SimpleMailMessage: to = request.getEmail(), subject = "Order Update — Order #" + request.getOrderId(), text = request.getMessage()
2. Call mailSender.send(...), wrapped in try/catch catching MailException (or a broad Exception if that's simpler) — if sending fails, log the failure clearly (log.error with the exception) but do NOT rethrow or let it propagate; a failed email must never turn into a 500 for the caller
3. Regardless of whether the send succeeded or failed, also log the notification at INFO level: the orderId, email, and message — this preserves the original v4 "logs order outcome messages" behavior even now that real email is being sent, so you always have a local record even if Gmail is unreachable

CONTROLLER (package: com.ecommerce.notificationservice.controller):
- Class: NotificationController, base path /api/notifications
- POST /api/notifications — body: NotificationRequest, @Valid — calls sendNotification, returns 201 with no body (or an empty body, e.g. ResponseEntity.status(201).build())
- No GlobalExceptionHandler needed for business logic — the one thing that could go wrong (mail sending) is already caught and logged inside the service, never thrown. Standard @Valid validation failures (missing/malformed fields) can fall through to Spring's default 400 behavior — this service doesn't need a custom GlobalExceptionHandler in v4, unlike every other backend service, since there's genuinely nothing else for one to catch.

CONSTRAINTS:
- Do not add retry, circuit breaker, or timeout logic anywhere — that's v5.
- Do not add caching anywhere.
- Do not add a Feign client of any kind.
- Do not add authentication/authorization (role checks) — that's v5.
- Do not add an HTML email template — plain SimpleMailMessage text is correct for this version; an HTML template is optional future polish, not required now.
- Do not add unit tests in this prompt — manual verification (an actual email arriving) is the real test here and can't be meaningfully mocked without losing the point of this version.
- Output every file in full: application.yml, NotificationRequest DTO, InternalSecretFilter, FilterConfig, NotificationService, NotificationController, and NotificationServiceApplication.java. Do NOT output pom.xml.

VERIFICATION (tell me how to confirm this works):
- Confirm notification-service registers with Eureka on startup.
- Send POST /api/notifications directly to port 8086 (with the correct X-Internal-Secret header) with a body like {"orderId": 1, "email": "YOUR_OWN_REAL_EMAIL_HERE", "message": "Test notification from v4"} — check that inbox and confirm the email actually arrives, with the subject and body matching what was sent.
- Check the console/log output and confirm the INFO log line appears regardless, matching the same orderId/email/message.
- Temporarily change the App Password in application.yml to something wrong, restart, resend the same request — confirm the endpoint still returns 201 (not a 500), and confirm the log shows an ERROR about the failed send rather than a silent failure.
- Restore the correct App Password afterward and confirm sending works again.
- Send the same request with NO X-Internal-Secret header — confirm 401, request never reaches the controller.
- Send a request with a malformed email field (e.g. "notanemail") — confirm 400 with a validation error message.