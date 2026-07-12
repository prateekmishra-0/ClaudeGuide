# PROMPT 5 — frontend-service, Part A (setup + browsing)

> Paste everything in the code block below directly into Amazon Q as a single prompt.
> This is Part A of two frontend prompts — this one covers project setup, product-service integration, and the two read-only browsing pages. Part B (Prompt 6) adds register/login/cart/checkout/history on top of this same project.
> product-service must already be built and running for the verification step, but this prompt itself doesn't need that to generate correctly.

```
Create a Spring Boot MVC + Thymeleaf web application from scratch called frontend-service. This is a server-rendered frontend with NO JavaScript — all interactivity is plain HTML forms and links, all pages rendered server-side via Thymeleaf.

PROJECT SETUP:
- Build tool: Maven
- Java version: 17
- Spring Boot version: 3.2.x
- Group: com.ecommerce
- Artifact: frontend-service
- Package: com.ecommerce.frontendservice
- Packaging: Jar

DEPENDENCIES (pom.xml):
- spring-boot-starter-web
- spring-boot-starter-thymeleaf
- spring-cloud-starter-netflix-eureka-client
- spring-cloud-starter-loadbalancer
- Spring Cloud version: 2023.0.x (compatible with Spring Boot 3.2.x) — add spring-cloud-dependencies BOM in dependencyManagement

MAIN APPLICATION CLASS:
- Class name: FrontendServiceApplication
- Location: com.ecommerce.frontendservice
- Annotations: @SpringBootApplication, @EnableDiscoveryClient

CONFIGURATION FILE:
- File: src/main/resources/application.yml
- server.port: 8080
- spring.application.name: frontend-service
- spring.thymeleaf.cache: false — so template edits reload without restarting during development
- eureka.client.service-url.defaultZone: http://localhost:8761/eureka

REST CLIENT BEAN CONFIGURATION (package: com.ecommerce.frontendservice.config):
- A @Configuration class exposing a @Bean of type RestClient.Builder, annotated @LoadBalanced, so calls resolve "product-service" through Eureka

DTOs (package: com.ecommerce.frontendservice.dto):
- ProductResponse: id (Long), name (String), description (String), price (BigDecimal), stockQuantity (Integer), categoryId (Long) — mirrors what product-service returns, used purely to deserialize the response, not persisted anywhere here

CLIENT SERVICE (package: com.ecommerce.frontendservice.client):
- Class: ProductServiceClient — uses the @LoadBalanced RestClient.Builder, base URL "http://product-service"
- Method: List<ProductResponse> getAllProducts() — calls GET /api/products; note this endpoint returns a Page<Product> from product-service, so deserialize accordingly and extract just the "content" list (map the raw JSON response, don't assume a flat array)
- Method: ProductResponse getProductById(Long id) — calls GET /api/products/{id}; if 404, throw our own ProductNotFoundException with a clear message

CONTROLLERS (package: com.ecommerce.frontendservice.controller):

HomeController
- GET / — calls getAllProducts, adds the list to the Model under attribute name "products", returns view name "home"

ProductController
- GET /products/{id} — calls getProductById, adds the result to the Model under attribute name "product", returns view name "product-detail"

ERROR HANDLING (package: com.ecommerce.frontendservice.exception):
- Custom exception: ProductNotFoundException extends RuntimeException (constructor takes a String message)
- Class: WebExceptionHandler, annotated @ControllerAdvice (note: @ControllerAdvice, NOT @RestControllerAdvice — this app renders HTML views, not JSON)
  - @ExceptionHandler(ProductNotFoundException.class) — add the exception's message to the Model under attribute "errorMessage", return view name "error" with HTTP status 404 (use @ResponseStatus or a ModelAndView with status set)
  - @ExceptionHandler(Exception.class) — generic fallback, add a generic message like "Something went wrong, please try again" to the Model under "errorMessage", return view name "error" with HTTP status 500

THYMELEAF TEMPLATES (src/main/resources/templates/):

home.html
- Simple page listing all products from the "products" model attribute
- For each product: display name, price, and a link to /products/{id} (use th:each and th:href with Thymeleaf link expressions, e.g. @{/products/{id}(id=${product.id})})
- Plain HTML structure — a heading, then a grid or list of product cards, each showing name, price, and a "View Details" link

product-detail.html
- Displays the single "product" model attribute's full details: name, description, price, stockQuantity
- Include a link back to home ("/")
- Leave a placeholder comment <!-- Add to cart form will go here in Part B --> where the add-to-cart form will be inserted in the next prompt — do not build the form itself yet

error.html
- Displays the "errorMessage" model attribute
- Include a link back to home ("/")

STATIC CSS (src/main/resources/static/css/style.css):
- Plain, minimal CSS — no framework, no build step. Style the product grid as a simple responsive grid (CSS Grid or Flexbox is fine), basic readable typography, a simple nav/header bar with the site name. Keep it clean and presentable but simple — this is not a design-heavy requirement.
- Link this stylesheet in a shared layout. Create a minimal common header fragment (src/main/resources/templates/fragments/header.html) using Thymeleaf's th:fragment, included via th:replace in home.html, product-detail.html, and error.html, so the <head> with the CSS link isn't duplicated three times.

CONSTRAINTS:
- Do not use any JavaScript anywhere, including inline event handlers — every interaction must be a plain <a> link or a plain HTML <form> with a standard HTTP method.
- Do not add authentication, session handling, cart, or checkout pages yet — those are Part B (the next prompt), which will extend this same project.
- Do not add a database or any datasource to this service — it has none, it only calls product-service.
- Output every file in full: pom.xml, application.yml, the RestClient config class, ProductResponse DTO, ProductServiceClient, both controllers, ProductNotFoundException, WebExceptionHandler, the header fragment, all three Thymeleaf templates, and style.css.

VERIFICATION (tell me how to confirm this works):
- Explain how to confirm, with product-service already running and containing at least 2-3 products: visiting http://localhost:8080/ shows the product grid, clicking a product's "View Details" link shows its detail page, and visiting a non-existent product id (e.g. /products/9999) shows the error page with a 404 status.
```
