# PROMPT 5 — frontend-service, Part A (setup + browsing)

> Paste everything in the code block below directly into Amazon Q as a single prompt.
> This is Part A of two frontend prompts — this one covers project setup, product-service integration via a Feign client, and the two read-only browsing pages. Part B (Prompt 6) adds register/login/cart/checkout/history on top of this same project.
> Assumes the project was already generated via Spring Initializr per the settings above.
> product-service must already be built and running for the verification step, but this prompt itself doesn't need that to generate correctly.

```
I have already generated this Spring Boot project via Spring Initializr with the following settings — do not modify, add, or remove anything in pom.xml, and do not change the Spring Boot or Spring Cloud version:

- Spring Boot 4.1.0, Spring Cloud release train 2025.1.2 (already configured in pom.xml)
- Group: com.ecommerce, Artifact: frontend-service, Package: com.ecommerce.frontendservice
- Dependencies present: Spring Web, Thymeleaf, Eureka Discovery Client, OpenFeign, Spring Cloud LoadBalancer
- Configuration format: application.yml (already exists, currently empty/default)
- This is a server-rendered frontend with NO JavaScript — all interactivity is plain HTML forms and links, all pages rendered server-side via Thymeleaf.

Your task is ONLY to write the application logic, configuration, and templates on top of this existing project. Do not touch pom.xml. Do not suggest adding any dependency beyond what's listed above.

MAIN APPLICATION CLASS:
- File: src/main/java/com/ecommerce/frontendservice/FrontendServiceApplication.java
- Annotations: @SpringBootApplication, @EnableDiscoveryClient, @EnableFeignClients

CONFIGURATION FILE:
- File: src/main/resources/application.yml
- server.port: 8080
- spring.application.name: frontend-service
- spring.thymeleaf.cache: false — so template edits reload without restarting during development
- eureka.client.service-url.defaultZone: http://localhost:8761/eureka

DTOs (package: com.ecommerce.frontendservice.dto):
- ProductResponse: id (Long), name (String), description (String), price (BigDecimal), stockQuantity (Integer), categoryId (Long) — mirrors what product-service returns, used purely to deserialize the response, not persisted anywhere here
- ProductPageResponse: content (List<ProductResponse>) — product-service's GET /api/products returns a Spring Data Page object, whose JSON shape has the actual list under a "content" key plus pagination metadata fields we don't need; this DTO only needs the "content" field, Jackson will ignore the rest of the page JSON automatically since we're not declaring those other fields

FEIGN CLIENT (package: com.ecommerce.frontendservice.client):
- Interface: ProductServiceClient, annotated @FeignClient(name = "product-service")
- Method: @GetMapping("/api/products") ProductPageResponse getAllProducts() — deserializes into ProductPageResponse; the caller (see controller below) extracts .getContent() to get the actual list
- Method: @GetMapping("/api/products/{id}") ProductResponse getProductById(@PathVariable("id") Long id)
- This is purely a declarative interface — no implementation body, no manual URL/host construction. Spring Cloud OpenFeign generates the implementation at runtime, resolving "product-service" through Eureka.
- Error handling: Feign throws FeignException subtypes on non-2xx responses. Do NOT write a custom ErrorDecoder — instead, catch feign.FeignException.NotFound specifically at the call site in HomeController/ProductController (see below) and translate it into our own ProductNotFoundException.

CONTROLLERS (package: com.ecommerce.frontendservice.controller):

HomeController
- GET / — calls productServiceClient.getAllProducts(), extracts the content list, adds it to the Model under attribute name "products", returns view name "home"

ProductController
- GET /products/{id} — calls productServiceClient.getProductById(id); catch feign.FeignException.NotFound and re-throw as our own ProductNotFoundException with a clear message; on success, adds the result to the Model under attribute name "product", returns view name "product-detail"

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
- Do not write a custom Feign ErrorDecoder — catching the specific FeignException.NotFound subtype at each call site is sufficient and correct for this version.
- Do not add authentication, session handling, cart, or checkout pages yet — those are Part B (the next prompt), which will extend this same project.
- Do not add a database or any datasource to this service — it has none, it only calls product-service.
- Output every file in full: application.yml, both DTOs, the ProductServiceClient Feign interface, both controllers, ProductNotFoundException, WebExceptionHandler, the header fragment, all three Thymeleaf templates, and style.css. Do NOT output pom.xml — it already exists and must not change.

VERIFICATION (tell me how to confirm this works):
- Explain how to confirm, with product-service already running and containing at least 2-3 products: visiting http://localhost:8080/ shows the product grid, clicking a product's "View Details" link shows its detail page, and visiting a non-existent product id (e.g. /products/9999) shows the error page with a 404 status.
```