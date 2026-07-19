This is NOT a new project — you're continuing in the existing recommendation-service project from v3. Do not run Spring Initializr again. No dependency changes are needed for this version — do not modify pom.xml at all, do not suggest any addition.

Existing files in this project (do not regenerate these from scratch — you are extending/modifying them):
- ProductResponse, OrderItemResponse, OrderResponse, ProductPageResponse DTOs — unchanged, do not touch
- OrderServiceClient — unchanged, do not touch (getOrderHistory, getOrdersContainingProduct stay exactly as they are)
- ProductServiceClient — EXTENDED below (one new method), existing methods (getProductById, getProductsByCategory) unchanged
- InternalSecretFilter, FilterConfig, FeignInternalSecretInterceptor — unchanged, do not touch
- RecommendationController — unchanged, do not touch. Both existing endpoint signatures (GET /api/recommendations/product/{productId}, GET /api/recommendations/user/{userId}) stay exactly as they are — only the underlying RecommendationService logic changes in this prompt.
- RecommendationServiceApplication — unchanged, do not touch

No configuration file change in this version — application.yaml stays exactly as it is from v3.

MODIFY: ProductServiceClient (package: com.ecommerce.recommendationservice.client)
- Add method: @GetMapping("/api/products/{id}/views/count-by-viewer") Map<Long, Long> getViewCountsByViewer(@PathVariable("id") Long id)
- This deserializes product-service's Map<userId, viewCount> response directly into Map<Long, Long> — Jackson handles the numeric-key conversion automatically, no custom deserializer needed.
- Purely declarative, no implementation body. Do not modify the two existing methods on this interface.

MODIFY: RecommendationService (package: com.ecommerce.recommendationservice.service)

Add new constants (private static final, alongside the existing RECOMMENDATION_COUNT = 4):
- PURCHASE_WEIGHT = 3
- VIEW_WEIGHT = 1
- CATEGORY_AFFINITY_BOOST = 2
- TOP_CATEGORIES_COUNT = 2

Add a new private helper method: Map<Long, Double> computeCombinedScores(Long productId)
1. Call orderServiceClient.getOrdersContainingProduct(productId), wrapped in try/catch treating any exception as an empty list (same fail-open pattern already used everywhere in this class)
2. From the returned orders' items, tally every OTHER productId's co-purchase count into a Map<Long, Integer> — this is exactly v3's existing Rule 1 tallying logic, reused as-is, not reimplemented differently
3. Call productServiceClient.getViewCountsByViewer(productId), wrapped in try/catch treating any exception as an empty map — call this ONCE for the original productId and reuse the result (its keySet is "distinct users who viewed X") rather than re-fetching it inside the loop below
4. For every productId Y that appears as a key in the co-purchase map from step 2: call productServiceClient.getViewCountsByViewer(Y), wrapped in try/catch treating any exception as an empty map — take its keySet ("distinct users who viewed Y"), intersect with X's viewer set from step 3, the intersection's size is co_view_count(X, Y)
5. For every Y in the co-purchase map, compute combinedScore = (coPurchaseCount(Y) * PURCHASE_WEIGHT) + (coViewCount(X,Y) * VIEW_WEIGHT), store in the returned Map<Long, Double>
6. Return this map — note it does NOT include category-fallback candidates, only genuine co-purchase candidates with a real combined score. Category fallback padding is applied by the caller, same as v3's Rule 2 was applied by the caller.
- Do not add tie-breaking logic beyond whatever natural ordering Map/stream operations give when two scores are exactly equal — same "no tie-breaking needed" stance as v3.

MODIFY: List<ProductResponse> getRecommendationsForProduct(Long productId)
1. Call computeCombinedScores(productId) to get the scored candidate map
2. Sort candidate productIds descending by score, take the top RECOMMENDATION_COUNT — this REPLACES v3's plain "sort by raw co-purchase count" step; v3's counting logic itself isn't removed, it's just no longer the final ranking signal on its own
3. If this produced fewer than RECOMMENDATION_COUNT results: apply the existing v3 Rule 2 category-fallback logic unchanged — resolve the original product's categoryId (try/catch, skip Rule 2 entirely on failure, same as v3), fetch same-category products, exclude the original product and anything already selected, fill remaining slots up to RECOMMENDATION_COUNT. Fallback-filled items have no combined score — they're appended after the scored results, in whatever order product-service returns them, same as v3.
4. Resolve each final productId to a full ProductResponse via productServiceClient.getProductById, skipping any individual resolution failure rather than failing the whole call — unchanged from v3.
5. Return the final list.

MODIFY: List<ProductResponse> getRecommendationsForUser(Long userId)
1. Call orderServiceClient.getOrderHistory(userId), wrapped in try/catch treating any exception as an empty list; if empty, return an empty list immediately — unchanged from v3.
2. NEW — compute the user's top categories: across ALL of the user's orders (not just the most recent one) and ALL items within them, tally a Map<Long categoryId, Integer purchaseCount>. Resolving each item's categoryId requires calling productServiceClient.getProductById(productId) for each distinct productId encountered (ProductResponse already carries categoryId, from v3) — wrap each individual lookup in try/catch, skip that item's contribution to the tally on failure rather than aborting. Sort descending by count, take the top TOP_CATEGORIES_COUNT category ids into a Set<Long>.
3. Take the most recent PLACED order (unchanged from v3 — the last element, or explicitly the one with the latest createdAt if you parse it, either is acceptable).
4. For each distinct productId in that order's items, call computeCombinedScores(thatProductId) — this gives you one Map<Long, Double> of scored candidates per base product in the order.
5. Merge these per-base-product maps into one overall Map<Long, Double> of candidate scores. Where the same candidate productId appears from more than one base product, keep the HIGHEST score encountered for it (not a sum) — this reflects the architecture's "score relative to the most relevant purchased product" phrasing, i.e. a candidate is judged by its best pairing, not accumulated across every base product. State this interpretation in a code comment since the architecture file doesn't spell out the merge rule explicitly.
6. Exclude any candidate productId the user has already purchased anywhere in their FULL order history (across all orders, not just the most recent) — unchanged rule from v3, just make sure it's checked against the full history, not just the most recent order.
7. NEW — apply the category-affinity boost: for each remaining candidate, resolve its categoryId (via productServiceClient.getProductById, try/catch, treat a failed resolution as "no boost" rather than skipping the candidate entirely — a missing boost is a minor scoring miss, not a reason to drop a candidate), add CATEGORY_AFFINITY_BOOST to its score if that categoryId is in the top-categories set from step 2.
8. Sort the final candidate map descending by (boosted) score, take the top RECOMMENDATION_COUNT.
9. Resolve each final productId to a full ProductResponse, skipping individual resolution failures, same pattern as everywhere else in this class.
10. Return the final list — never throw; any failure anywhere in this flow results in as complete a list as could be assembled, or an empty list in the worst case.

CONSTRAINTS:
- Do not add a database, entity, or repository anywhere — this service remains stateless, full stop.
- Do not add caching of any kind — recomputing from live calls on every request is still the deliberate behavior in this version.
- Do not add retry, circuit breaker, or timeout configuration on any Feign client — that's v5.
- Do not change v3's actual co-purchase tallying algorithm itself (the map-building logic in step 2 of computeCombinedScores) — it's being reused, not reimplemented, so any existing correct logic from v3 should carry over unchanged into this helper method.
- Do not add any new endpoint — the two existing ones are unchanged.
- Do not add unit tests in this prompt — same discipline as v3's prompt: manual verification via the steps below is sufficient for this version.
- Output every file that actually changes or is new: the modified ProductServiceClient (full file) and the modified RecommendationService (full file). Do not regenerate or restate application.yaml, any DTO, OrderServiceClient, InternalSecretFilter, FilterConfig, FeignInternalSecretInterceptor, RecommendationController, or RecommendationServiceApplication.

VERIFICATION (tell me how to confirm this works):
- Prerequisite: eureka-server, product-service, order-service, and recommendation-service all running and registered; you'll need a product with both real order history (from earlier testing) and some recorded views (from product-service's v4 view-tracking endpoint), plus at least one other product sharing views with it from overlapping users.
- Send GET /api/recommendations/product/{id} for a product with both purchase and view history — confirm the ranking now reflects the combined formula: manually compute co_purchase*3 + co_view*1 for a couple of candidates yourself (querying order-service's containing-product and product-service's count-by-viewer endpoints directly) and confirm the returned order matches your hand calculation.
- Generate several extra views (via POST /api/products/{id}/view) for a product that currently ranks lower in a recommendation list, then re-send the same GET /api/recommendations/product/{id} request — confirm its ranking visibly improves, proving view weighting actually changes the outcome, not just the purchase count from v3.
- Send GET /api/recommendations/user/{userId} for a user who has purchased from a category multiple times — confirm products from that category tend to rank higher than similarly-co-purchased/co-viewed products from other categories, reflecting the +2 category-affinity boost.
- Manually stop product-service (keep order-service up), re-send GET /api/recommendations/product/{id} — confirm it still returns an empty list, not a 500 (view-count and category lookups failing should degrade gracefully, same fail-open principle as v3).
- Confirm both endpoints still reject requests with no X-Internal-Secret header with a 401, unchanged from v3.