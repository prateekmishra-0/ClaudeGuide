# PROMPT 2 (FINAL) — product-service

> Paste everything in the code block below directly into Amazon Q as a single prompt.
> This service is self-contained — it does not call any other service.
> Assumes the project was already generated via Spring Initializr per the settings given earlier (Spring Boot 4.1.0, Group com.ecommerce, Artifact product-service, Package com.ecommerce.productservice, dependencies: Spring Web, Spring Data JPA, Validation, Eureka Discovery Client, PostgreSQL Driver, Lombok).
> This version folds in the duplicate-category fix and targeted unit tests from the start — nothing further needs to be patched on afterward.

```
I have already generated this Spring Boot project via Spring Initializr — do not modify, add, or remove anything in pom.xml, and do not change the Spring Boot or Spring Cloud version:

- Spring Boot 4.1.0, Spring Cloud release train 2025.1.2 (already configured in pom.xml)
- Group: com.ecommerce, Artifact: product-service, Package: com.ecommerce.productservice
- Dependencies present: Spring Web, Spring Data JPA, Validation, Eureka Discovery Client, PostgreSQL Driver, Lombok (spring-boot-starter-test is included by default, no action needed for it)
- Configuration format: application.yml (already exists, currently empty/default)

Your task is ONLY to write the application logic, configuration, and tests on top of this existing project. Do not touch pom.xml. Do not suggest adding any dependency beyond what's listed above.

MAIN APPLICATION CLASS:
- File: src/main/java/com/ecommerce/productservice/ProductServiceApplication.java
- Annotations: @SpringBootApplication, @EnableDiscoveryClient

CONFIGURATION FILE:
- File: src/main/resources/application.yaml
- server.port: 8081
- spring.application.name: product-service
- spring.datasource.url: jdbc:postgresql://localhost:5432/product_db
- spring.datasource.username: postgres
- spring.datasource.password: root
- spring.jpa.hibernate.ddl-auto: update
- spring.jpa.show-sql: true
- eureka.client.service-url.defaultZone: http://localhost:8761/eureka

ENTITIES (package: com.ecommerce.productservice.entity):

Entity: Category
- id: Long, @Id, @GeneratedValue(strategy = GenerationType.IDENTITY)
- name: String, @NotBlank, @Column(unique = true, length = 60)
- description: String, @Column(length = 255), nullable

Entity: Product
- id: Long, @Id, @GeneratedValue(strategy = GenerationType.IDENTITY)
- name: String, @NotBlank, @Column(length = 100)
- description: String, @Column(length = 500), nullable
- price: BigDecimal, @NotNull, @DecimalMin(value = "0.01")
- stockQuantity: Integer, @NotNull, @Min(0)
- categoryId: Long, @NotNull — plain column, NOT a @ManyToOne relationship, no object navigation to Category
- createdAt: LocalDateTime — set only inside a @PrePersist method, must never be settable via the JSON request body (use @Column(updatable = false) and set it in @PrePersist)

Use Lombok @Getter @Setter @NoArgsConstructor @AllArgsConstructor on both entities. Do not add a @Builder unless needed elsewhere in the code you generate.

REPOSITORIES (package: com.ecommerce.productservice.repository):
- CategoryRepository extends JpaRepository<Category, Long>
  - Add method: boolean existsByName(String name) — used to reject duplicate category names before saving (see CategoryController below)
- ProductRepository extends JpaRepository<Product, Long>
  - Add method: Page<Product> findByCategoryId(Long categoryId, Pageable pageable)

CONTROLLERS (package: com.ecommerce.productservice.controller):

CategoryController — base path /api/categories
- GET /api/categories — return List<Category>, all categories
- POST /api/categories — @Valid @RequestBody; BEFORE saving, call categoryRepository.existsByName(request's name); if true, throw CategoryAlreadyExistsException with a message like "Category with name '<name>' already exists"; if false, proceed to save and return 201 with the created entity

ProductController — base path /api/products
- GET /api/products — return Page<Product>; accepts optional query param categoryId (Long); if present, filter by category using the repository method above; supports standard Pageable query params (page, size, sort)
- GET /api/products/{id} — return the product; throw ProductNotFoundException if not found, do not return null or Optional directly from the controller
- POST /api/products — @Valid @RequestBody, create, return 201 with the created entity
- PUT /api/products/{id} — @Valid @RequestBody, full replace of an existing product's fields (not createdAt), 404 via ProductNotFoundException if the id doesn't exist
- DELETE /api/products/{id} — hard delete, return 204, 404 via ProductNotFoundException if the id doesn't exist
- PUT /api/products/{id}/stock — request body: a simple record/DTO with a single field "delta" (Integer, can be negative or positive); apply stockQuantity += delta; if the resulting stockQuantity would be negative, throw InsufficientStockException and do not save the change; otherwise save and return the updated product. Use @PutMapping, not @PatchMapping — this project's default Feign client does not support PATCH (throws a ProtocolException at runtime when order-service calls it in v2), and PUT avoids that without any new dependency.

EXCEPTION HANDLING (package: com.ecommerce.productservice.exception):
- Custom exception: ProductNotFoundException extends RuntimeException (constructor takes a String message)
- Custom exception: CategoryNotFoundException extends RuntimeException (constructor takes a String message)
- Custom exception: InsufficientStockException extends RuntimeException (constructor takes a String message)
- Custom exception: CategoryAlreadyExistsException extends RuntimeException (constructor takes a String message)
- Class: GlobalExceptionHandler, annotated @RestControllerAdvice
  - @ExceptionHandler(MethodArgumentNotValidException.class) → 400, return a body listing each invalid field and its validation message
  - @ExceptionHandler(ProductNotFoundException.class) → 404, return { "message": <exception message> }
  - @ExceptionHandler(CategoryNotFoundException.class) → 404, return { "message": <exception message> }
  - @ExceptionHandler(InsufficientStockException.class) → 409, return { "message": <exception message> }
  - @ExceptionHandler(CategoryAlreadyExistsException.class) → 409, return { "message": <exception message> }
  - @ExceptionHandler(org.springframework.dao.DataIntegrityViolationException.class) → 409, return { "message": "A record with this name already exists" } — this is a backstop for the rare race condition where two requests pass the existsByName check at nearly the same instant; it should almost never trigger in normal testing, but must be present
  - @ExceptionHandler(Exception.class) → 500, generic fallback, return { "message": "An unexpected error occurred" }

UNIT TESTS (package: com.ecommerce.productservice.controller, in src/test/java):

Test class: ProductControllerTest
- Annotated @WebMvcTest(ProductController.class) — loads only the web layer, not the full application context or a real database
- Mock ProductRepository with @MockBean
- Use MockMvc to send requests and assert on HTTP status + JSON body
1. updateStock_whenResultingQuantityWouldBeNegative_returns409 — mock a Product with stockQuantity = 5, send PUT /api/products/{id}/stock with {"delta": -10}, assert status 409 and that the response body's "message" field is present and non-empty
2. updateStock_whenResultingQuantityIsValid_returns200AndUpdatedQuantity — mock a Product with stockQuantity = 5, send PUT with {"delta": -2}, assert status 200 and that the returned product's stockQuantity is 3
3. getProduct_whenIdDoesNotExist_returns404 — mock the repository to return Optional.empty(), send GET /api/products/9999, assert status 404
4. createProduct_whenValid_returns201WithCreatedEntity — mock the repository's save() to return a Product with an assigned id, send a valid POST body, assert status 201 and that the response contains the expected fields

Test class: CategoryControllerTest
- Annotated @WebMvcTest(CategoryController.class)
- Mock CategoryRepository with @MockBean
5. createCategory_whenNameAlreadyExists_returns409 — mock existsByName to return true, send POST /api/categories with any name, assert status 409, assert the response body's "message" mentions the category already exists
6. createCategory_whenNameIsNew_returns201 — mock existsByName to return false and save() to return a Category with an assigned id, send a valid POST body, assert status 201

CONSTRAINTS:
- Do not add a Category → Product @OneToMany relationship or any JPA object-level relationship anywhere in this service — categoryId stays a plain Long column as specified.
- Do not add Spring Security, Swagger/OpenAPI, or suggest any dependency beyond what's already present in pom.xml.
- Do not add a service/business layer beyond what's implied above — controllers may call repositories directly for this version.
- Do not add any endpoints beyond the ones listed above.
- Do not add DTOs unless explicitly specified above (the stock PATCH body is the one exception — that needs a small request DTO/record).
- Do not add a similar existsBy check to Product — Product has no unique fields in this version, the duplicate-check pattern is scoped to Category only.
- Do not add @SpringBootTest, Testcontainers, or a real Postgres connection anywhere in the tests — @WebMvcTest with mocked repositories is correct and sufficient.
- Output every file in full: application.yml, both entities, both repositories, both controllers, all four custom exception classes, GlobalExceptionHandler, and both test classes (ProductControllerTest, CategoryControllerTest). Do NOT output pom.xml — it already exists and must not change.

IMPORTANT — Spring Boot 4.1.0 import paths: use exactly these two imports in the test class, nothing else:
import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
Do NOT use org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest or org.springframework.boot.test.mock.mockito.MockBean — both are Spring Boot 3.x-era paths that no longer exist in 4.1.0.

VERIFICATION (tell me how to confirm this works):
- List the exact curl or Postman requests to test: create a category, create 2-3 products under it, list products, get one by id, update one, PUT its stock down until InsufficientStockException triggers (409), delete one, then attempt to create a category with a name that already exists and confirm it returns 409 with a clear message (not a generic 500).
- Also tell me the exact Maven command to run just the two new test classes, and what a fully-passing console output should look like.
```