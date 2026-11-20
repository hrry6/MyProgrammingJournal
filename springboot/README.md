# 🍃 Spring Boot

Structured backend engineering notes, production blueprints, and reusable patterns for building Spring Boot 3+ REST APIs with Java 21 and PostgreSQL.

[⬅️ Back to Home](../README.md)

---

## 📑 Notes

| Notes | Description |
| :--- | :--- |
| [Project Init & Maven Wrapper (`mvnw`)](./project-init-and-mvnw.md) | `spring init` CLI flags, lifecycle commands, and `./mvnw` options |
| [`pom.xml` & Core Dependencies](./pom-and-dependencies.md) | Dependency breakdown, Maven scopes, JJWT, Lombok, and Testcontainers setup |
| [Application Properties & Profiles](./application-properties.md) | `application.properties`, `${VAR:default}` placeholders, and `application-test.properties` |
| [Architecture & Core Concepts](./architecture-and-concepts.md) | Layered vs. Modular Monolith, microservice migration path, IoC/DI, and request lifecycle |
| [Internals & General Annotations](./internals-and-general-annotations.md) | `@SpringBootApplication`, auto-configuration, AOP proxies, and cross-layer annotations |
| [Security & JWT Configuration](./security-and-jwt-config.md) | Stateless session policy, `JwtService` (JJWT 0.12.6), `JwtAuthenticationFilter`, `SecurityFilterChain`, and CORS |
| [Common Utilities & Exception Handling](./common-and-exception-handling.md) | `ApiResponse<T>`, custom exceptions, `@RestControllerAdvice`, and JPA auditing (`BaseEntity`) |
| [JPA Entities & Relationships](./jpa-entities.md) | PostgreSQL entity mapping, `@ManyToOne` lazy loading, `UserDetails`, and Lombok gotchas |
| [DTOs & Bean Validation](./dtos-and-validation.md) | Immutable Java `record` DTOs, Jakarta Validation (`@Valid`), and Entity -> DTO mapping |
| [Repositories & JPA Queries](./repositories-and-jpa-queries.md) | Derived queries, `Page` vs. `Slice`, `@Query`, solving the N+1 problem, and `@Lock` concurrency control |
| [Service Layer & Transactions](./services-and-transactions.md) | Business logic encapsulation, `@Transactional` rules, and Hibernate dirty checking |
| [Controllers & REST API Design](./controllers-and-rest-api.md) | `@RestController` conventions, pagination, `@AuthenticationPrincipal`, `@PreAuthorize`, and HTTP status codes |
| [Unit & Integration Testing](./testing-unit-and-integration.md) | JUnit 5 + Mockito service tests, Testcontainers (`@ServiceConnection`), and `MockMvc` RBAC tests |
| [File Upload & Storage](./file-upload-and-storage.md) | Multipart uploads, MIME validation, path traversal protection, and serving files |
| [Microservices Architecture](./microservices-architecture.md) | API Gateway, `RestClient` / `@HttpExchange`, Resilience4j circuit breakers, and Transactional Outbox |
| [Microservice Folder Structure](./microservice-folder-structure.md) | Monorepo workspace layout, internal microservice packages, and sharing `.proto` contracts |
| [gRPC in Spring Boot](./grpc-in-spring-boot.md) | Protocol Buffers (`proto3`), Maven `protoc` setup, `@GrpcService` server, and `@GrpcClient` consumer |

---

## 🌲 Reference Project Structure (Modular Monolith)

```text
src
├── main
│   ├── java/com/hrry6/app
│   │   ├── config/
│   │   │   ├── JpaAuditingConfig.java
│   │   │   └── security/
│   │   │       ├── ApplicationAuthConfig.java
│   │   │       ├── JwtAuthenticationFilter.java
│   │   │       ├── JwtService.java
│   │   │       └── SecurityConfig.java
│   │   ├── common/
│   │   │   ├── dto/ApiResponse.java
│   │   │   ├── entity/BaseEntity.java
│   │   │   └── exception/
│   │   │       ├── DuplicateResourceException.java
│   │   │       ├── GlobalExceptionHandler.java
│   │   │       └── ResourceNotFoundException.java
│   │   └── modules/
│   │       ├── auth/          # AuthController, AuthService, UserRepository, User, dto/
│   │       ├── catalog/       # BookController, BookService, BookRepository, Book, dto/
│   │       └── loan/          # LoanController, LoanService, LoanRepository, Loan, dto/
│   └── resources/
│       └── application.properties
└── test
    ├── java/com/hrry6/app
    │   ├── AbstractIntegrationTest.java
    │   └── modules/catalog/
    │       ├── BookServiceTest.java
    │       └── BookIntegrationTest.java
    └── resources/
        └── application-test.properties
```
