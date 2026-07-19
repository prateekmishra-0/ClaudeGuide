This is NOT a new project — you're continuing in the existing product-service project from v1/v2. Do not run Spring Initializr again. No dependency changes are needed for this version — do not modify pom.xml at all, do not suggest any addition.

Existing files in this project (do not regenerate these from scratch — you are extending/modifying them):
- Category, Product entities — unchanged, do not touch
- CategoryRepository, ProductRepository — unchanged, do not touch
- StockUpdateRequest DTO — unchanged, do not touch
- CategoryController — unchanged, do not touch
- ProductController — EXTENDED below (two new endpoints), no existing method changes. If this project has business logic living directly in ProductController (calling repositories directly, no separate service class) — add the new logic there too, following the same pattern already used by its existing methods. If a separate ProductService class already exists in this project, put the new logic there instead and wire the controller to call it, matching whatever pattern is already established.
- ProductNotFoundException, CategoryNotFoundException, InsufficientStockException, CategoryAlreadyExistsException — unchanged, do not touch
- GlobalExceptionHandler — unchanged, do not touch, no new exception type is introduced in this version
- InternalSecretFilter, FilterConfig (from v2) — unchanged, do not touch

No configuration file change in this version — application.yml stays exactly as it is from v2.

NEW ENTITY (package: com.ecommerce.productservice.entity):
- Class: ProductView
    - id: Long, PK, auto-generated (IDENTITY)
    - userId: Long, @NotNull — plain reference only, no relationship object, same treatment as every cross-domain user reference elsewhere in this project
    - productId: Long, @NotNull — plain FK column, NOT a @ManyToOne relationship object. This deliberately follows the same pattern already established by Product.categoryId in this same service ("plain FK column, no @ManyToOne object mapping needed... keep it a raw ID, avoid JPA relationship complexity until it's actually needed"), for consistency with this codebase's own convention.
    - viewedAt: LocalDateTime, set once in a @PrePersist method, not client-settable
      Use Lombok @Getter @Setter @NoArgsConstructor @AllArgsConstructor on this entity, same as Category and Product elsewhere in this same service.
- No uniqueness constraint on any combination of fields — the same user viewing the same product twice must create two separate rows, since view frequency itself is a signal recommendation-service will count later. Do not add @Column(unique = true) anywhere on this entity, and do not add any pre-save existence check before inserting a new ProductView row.

NEW REPOSITORY (package: com.ecommerce.productservice.repository):
- Interface: ProductViewRepository extends JpaRepository<ProductView, Long>
- One custom query method, using a JPQL projection to get view counts grouped by viewer for a single product:
  @Query("SELECT pv.userId AS userId, COUNT(pv) AS viewCount FROM ProductView pv WHERE pv.productId = :productId GROUP BY pv.userId")
  List<ViewCountProjection> countViewsByUserForProduct(@Param("productId") Long productId);
- Define a small nested/separate interface projection in the same package:
  public interface ViewCountProjection {
  Long getUserId();
  Long getViewCount();
  }
- Do not add any other repository method.

NEW DTO (package: com.ecommerce.productservice.dto):
- ViewRequest: userId (Long, @NotNull) — request body for POST /api/products/{id}/view

MODIFY: ProductController (package: com.ecommerce.productservice.controller)

Add TWO new endpoints:

1. POST /api/products/{id}/view
    - body: ViewRequest, @Valid
    - First confirm the product exists: call productRepository.findById(id), throw the existing ProductNotFoundException if not found — same pattern every other single-product endpoint in this controller already uses
    - Create and save a new ProductView with productId = the path {id}, userId = request.getUserId() (viewedAt is set automatically via @PrePersist)
    - Return 201 with no meaningful body (e.g. ResponseEntity.status(HttpStatus.CREATED).build())

2. GET /api/products/{id}/views/count-by-viewer
    - Call productViewRepository.countViewsByUserForProduct(id)
    - Convert the resulting List<ViewCountProjection> into a Map<Long, Long> (userId → viewCount) — a plain in-memory conversion, e.g. via Collectors.toMap
    - Return 200 with the Map directly as the response body (Jackson serializes a Map<Long,Long> as a plain JSON object with string keys, e.g. {"3": 5, "7": 2} — this is expected and fine, recommendation-service's client-side deserialization will handle it)
    - Do not validate that the product exists first here — if the productId has zero views (new product, or a nonexistent id), an empty map is the correct, expected response, not a 404. This endpoint is called internally by recommendation-service, and an empty result must never be an error.

Do not modify any existing endpoint method in this controller.

CONSTRAINTS:
- Do not add authentication/authorization checks anywhere beyond what already exists (the internal-secret filter already covers both new endpoints automatically, since it has no whitelist).
- Do not add pagination to the count-by-viewer endpoint — a plain Map is correct for this version's scale.
- Do not touch InternalSecretFilter or FilterConfig — this is purely additive, the existing mechanism already covers these new endpoints with zero changes.
- Do not add a uniqueness constraint or existence-check anywhere on ProductView — repeat views by the same user for the same product are intentional, expected data.
- Do not add unit tests in this prompt — manual verification via the steps below is sufficient for this version.
- Output only the files that actually change or are new: application.yml is NOT changing so do not output it; output the new ProductView entity (full file), the new ProductViewRepository (full file, including the ViewCountProjection interface), the new ViewRequest DTO (full file), and the modified ProductController (full file). Do not regenerate or restate Category, Product, CategoryRepository, ProductRepository, StockUpdateRequest, CategoryController, any exception, GlobalExceptionHandler, InternalSecretFilter, or FilterConfig.

VERIFICATION (tell me how to confirm this works):
- With product-service running standalone (no other service needs to be up for these two endpoints specifically): send POST /api/products/{id}/view (with the correct X-Internal-Secret header) for a real product id, with a few different userId values, including sending the SAME userId for the SAME product twice — confirm each call returns 201 and confirm in the database that TWO separate rows exist for that repeated userId+productId pair, not one row with an incremented count.
- Send POST /api/products/{id}/view for a productId that doesn't exist — confirm 404 via the existing ProductNotFoundException, not a 500.
- Send GET /api/products/{id}/views/count-by-viewer for the product you just recorded views for — confirm the returned map's counts match what you actually sent (e.g. if you sent userId 5 twice and userId 7 once, confirm {"5": 2, "7": 1}).
- Send GET /api/products/{id}/views/count-by-viewer for a product with zero recorded views — confirm it returns an empty map {} with 200, not a 404 or error.
- Send either endpoint with no X-Internal-Secret header — confirm 401, request never reaches the controller.
- Confirm all existing product-service endpoints (categories, product CRUD, stock adjustment) still behave exactly as before — this change should be purely additive with zero regression.