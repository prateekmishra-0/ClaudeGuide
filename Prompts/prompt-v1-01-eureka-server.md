I have already generated this Spring Boot project via Spring Initializr with the following settings — do not modify, add, or remove anything in pom.xml, and do not change the Spring Boot or Spring Cloud version:

- Spring Boot 4.1.0, Spring Cloud release train 2025.1.2 (already configured in pom.xml)
- Group: com.ecommerce, Artifact: eureka-server, Package: com.ecommerce.eurekaserver
- Dependency present: spring-cloud-starter-netflix-eureka-server
- Configuration format: application.yml (already exists, currently empty/default)

Your task is ONLY to write the application logic on top of this existing project. Do not touch pom.xml. Do not suggest adding any dependency.

MAIN APPLICATION CLASS:
- File: src/main/java/com/ecommerce/eurekaserver/EurekaServerApplication.java
- Annotations: @SpringBootApplication, @EnableEurekaServer
- Standard main() method calling SpringApplication.run()

CONFIGURATION FILE:
- File: src/main/resources/application.yml
- server.port: 8761
- spring.application.name: eureka-server
- eureka.client.register-with-eureka: false
- eureka.client.fetch-registry: false
- eureka.server.enable-self-preservation: false
- eureka.instance.hostname: localhost

CONSTRAINTS:
- Do not add any REST controllers, entities, repositories, or business logic — this application's only purpose is to run the Eureka registry.
- Do not add Spring Security, Actuator, or any dependency-requiring feature.
- Output the full EurekaServerApplication.java and the full application.yml. Nothing else.

VERIFICATION (tell me how to confirm this works):
- Explain how to confirm the Eureka dashboard is reachable at http://localhost:8761 and what a successful startup looks like in the console logs.