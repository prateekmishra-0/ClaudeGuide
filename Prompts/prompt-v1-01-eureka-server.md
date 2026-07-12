# PROMPT 1 — eureka-server

> Paste everything in the code block below directly into Amazon Q as a single prompt.

```
Create a Spring Boot Eureka Server application from scratch.

PROJECT SETUP:
- Build tool: Maven
- Java version: 17
- Spring Boot version: 3.2.x
- Group: com.ecommerce
- Artifact: eureka-server
- Package: com.ecommerce.eurekaserver
- Packaging: Jar

DEPENDENCIES (pom.xml):
- spring-cloud-starter-netflix-eureka-server
- Spring Cloud version: 2023.0.x (compatible with Spring Boot 3.2.x) — add spring-cloud-dependencies BOM in dependencyManagement

MAIN APPLICATION CLASS:
- Class name: EurekaServerApplication
- Location: com.ecommerce.eurekaserver
- Annotations: @SpringBootApplication, @EnableEurekaServer
- Standard main() method calling SpringApplication.run()

CONFIGURATION FILE:
- File: src/main/resources/application.yml (YAML format, not .properties)
- server.port: 8761
- spring.application.name: eureka-server
- eureka.client.register-with-eureka: false
- eureka.client.fetch-registry: false
- eureka.server.enable-self-preservation: false (so deregistration of down services is immediate during local dev/testing, not delayed)
- eureka.instance.hostname: localhost

CONSTRAINTS:
- Do not add any REST controllers, entities, repositories, or business logic — this application's only purpose is to run the Eureka registry.
- Do not add spring-boot-starter-web as a separate dependency — it is already transitively included by eureka-server.
- Do not add Spring Security, Actuator, or any other dependency beyond what is listed above.
- Do not add a database, JPA, or any datasource configuration.
- Output the full pom.xml, the full EurekaServerApplication.java, and the full application.yml. Nothing else.

VERIFICATION (tell me how to confirm this works):
- After running the application, explain how to confirm the Eureka dashboard is reachable at http://localhost:8761 and what a successful startup looks like in the console logs.
```
