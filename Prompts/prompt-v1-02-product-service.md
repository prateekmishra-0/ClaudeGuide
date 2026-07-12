# PROMPT 2 — product-service

> Paste everything in the code block below directly into Amazon Q as a single prompt.
> This service is self-contained — it does not call any other service.

```
Create a Spring Boot microservice from scratch called product-service.

PROJECT SETUP:
- Build tool: Maven
- Java version: 17
- Spring Boot version: 3.2.x
- Group: com.ecommerce
- Artifact: product-service
- Package: com.ecommerce.productservice
- Packaging: Jar

DEPENDENCIES (pom.xml):
- spring-boot-starter-web
- spring-boot-starter-data-jpa
- spring-boot-starter-validation
- spring-cloud-starter-netflix-eureka-client
- Spring Cloud version: 2023.0.x (compatible with Spring Boot 3.2.x) — add spring-cloud-dependencies BOM in dependencyManagement
- postgresql (runtime scope, official PostgreSQL JDBC driver)
- lombok (optional scope)

MAIN APPLICATION CLASS:
- Class name: ProductServiceApplication
- Location: com.ecommerce.productservice
- Annotations: @SpringBootApplication, @EnableDiscoveryClient

CONFIGURATION FILE:
- File: src/main/resources/application.yml
- server.port: 8081
- spring.application.name: product-service
- spring.datasource.url: jdbc:postgresql://localhost:5432/ecommerce_db?currentSchema=product_schema
- spring.datasource.username: postgres
- spring.datasource.password: postgres
- spring.jpa.hibernate.ddl-auto: update
- spring.jpa.properties.hibernate.default_schema: product_schema
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
- createdAt: LocalDateTime — set only inside a @PrePersist method, must never be settable via the JSON request body (exclude it from any constructor/setter path a client could reach — use @Column(updatable = false) and set it in @PrePersist)

Use Lombok @Getter @Setter @NoArgsConstructor @AllArgsConstructor on both entities. Do not add a @Builder unless needed elsewhere in the code you generate.

REPOSITORIES (package: com.ecommerce.productservice.repository):
- CategoryRepository extends JpaRepository<Category, Long>
- ProductRepository extends JpaRepository<Product, Long>
  - Add method: Page<Product> findByCategoryId(Long categoryId, Pageable pageable)

CONTROLLERS (package: com.ecommerce.productservice.controller):

CategoryController — base path /api/categories
- GET /api/categories — return List<Category>, all categories
- POST /api/categories — create a category, @Valid @RequestBody, return 201 with the created entity

ProductController — base path /api/products
- GET /api/products — return Page<Product>; accepts optional query param categoryId (Long); if present, filter by category using the repository method above; supports standard Pageable query params (page, size, sort)
- GET /api/products/{id} — return the product; throw ProductNotFoundException if not found, do not return null or Optional directly from the controller
- POST /api/products — @Valid @RequestBody, create, return 201 with the created entity
- PUT /api/products/{id} — @Valid @RequestBody, full replace of an existing product's fields (not createdAt), 404 via ProductNotFoundException if the id doesn't exist
- DELETE /api/products/{id} — hard delete, return 204, 404 via ProductNotFoundException if the id doesn't exist
- PATCH /api/products/{id}/stock — request body: a simple record/DTO with a single field "delta" (Integer, can be negative or positive); apply stockQuantity += delta; if the resulting stockQuantity would be negative, throw a new custom exception InsufficientStockException and do not save the change; otherwise save and return the updated product

EXCEPTION HANDLING (package: com.ecommerce.productservice.exception):
- Custom exception: ProductNotFoundException extends RuntimeException (constructor takes a String message)
- Custom exception: CategoryNotFoundException extends RuntimeException (constructor takes a String message)
- Custom exception: InsufficientStockException extends RuntimeException (constructor takes a String message)
- Class: GlobalExceptionHandler, annotated @RestControllerAdvice
  - @ExceptionHandler(MethodArgumentNotValidException.class) → 400, return a body listing each invalid field and its validation message
  - @ExceptionHandler(ProductNotFoundException.class) → 404, return { "message": <exception message> }
  - @ExceptionHandler(CategoryNotFoundException.class) → 404, return { "message": <exception message> }
  - @ExceptionHandler(InsufficientStockException.class) → 409, return { "message": <exception message> }
  - @ExceptionHandler(Exception.class) → 500, generic fallback, return { "message": "An unexpected error occurred" }

CONSTRAINTS:
- Do not add a Category → Product @OneToMany relationship or any JPA object-level relationship anywhere in this service — categoryId stays a plain Long column as specified.
- Do not add Spring Security, Swagger/OpenAPI, or any dependency beyond what is listed above.
- Do not add a service/business layer beyond what's implied above — controllers may call repositories directly for this version.
- Do not add any endpoints beyond the ones listed above.
- Do not add DTOs unless explicitly specified above (the stock PATCH body is the one exception — that needs a small request DTO/record).
- Output every file in full: pom.xml, application.yml, both entities, both repositories, both controllers, both custom exception classes plus InsufficientStockException, and GlobalExceptionHandler.

VERIFICATION (tell me how to confirm this works):
- List the exact curl or Postman requests to test: create a category, create 2-3 products under it, list products, get one by id, update one, PATCH its stock down until InsufficientStockException triggers, then delete one.
```
