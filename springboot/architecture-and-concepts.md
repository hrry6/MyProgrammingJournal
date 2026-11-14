# Architecture & Core Concepts

[🏠 Home](../README.md) > [🍃 Spring Boot](./README.md)

---

## Architectural Patterns

### A. Layered Architecture (Package-by-Layer)

Organizes code strictly by technical responsibility. Best for small CRUD apps, prototypes, or MVPs.

```text
com.hrry6.app
├── config/          # Security, CORS, Beans
├── controller/      # HTTP endpoints (BookController, UserController)
├── service/         # Business logic (BookService, UserService)
├── repository/      # Database access (BookRepository, UserRepository)
├── model/           # JPA Entities (Book, User)
├── dto/             # Request & Response payloads
└── exception/       # Global error handlers
```

- **Pros:** Simple to understand, low initial boilerplate.
- **Cons:** Tight coupling across domains. Adding one feature requires touching 5+ folders; becomes messy as the codebase grows.

---

### B. Modular Monolith (Package-by-Feature)

Organizes code by **business domain**, keeping each module's controller, service, repository, and DTOs self-contained inside a single deployable JAR.

```text
com.hrry6.app
├── common/                  # Shared utilities, base entities, global exceptions
├── config/                  # App-wide Security, OpenAPI, Jackson configs
└── modules/
    ├── auth/                # Auth & User domain
    │   ├── AuthController.java
    │   ├── AuthService.java
    │   ├── UserRepository.java
    │   ├── User.java
    │   └── dto/
    ├── catalog/             # Book domain
    │   ├── BookController.java
    │   ├── BookService.java
    │   ├── BookRepository.java
    │   ├── Book.java
    │   └── dto/
    └── loan/                # Borrowing transaction domain
        ├── LoanController.java
        ├── LoanService.java
        ├── LoanRepository.java
        ├── Loan.java
        └── dto/
```

- **Pros:** High cohesion, low coupling, single database/deployment, easy refactoring.
- **Rule of Thumb:** Module A should call Module B's **Service** (or publish an internal event), never directly access Module B's **Repository**.

---

### C. Transitioning to Microservices

Starting with a **Modular Monolith** is the safest path toward microservices—avoid splitting into microservices prematurely before domain boundaries are clear.

| Stage | Structure | Database | Communication |
| :--- | :--- | :--- | :--- |
| **1. Modular Monolith** | Isolated feature packages in 1 JAR | Single DB (logically separated tables) | In-memory Java method calls / Spring Events |
| **2. Database Isolation** | Same single JAR, strict module boundaries | Separate schemas or DBs per module (no cross-module SQL joins) | Service interfaces / Domain events |
| **3. Microservices** | Extract heavy/independent modules into separate Spring Boot apps | Dedicated DB per service | REST / gRPC (sync) or RabbitMQ / Kafka (async) |

**Migration Checklist When Extracting a Module:**

1. Remove direct JPA entity relationships (`@ManyToOne`) across modules—reference foreign modules by **ID only** (e.g., `UUID userId` instead of `User user`).
2. Replace direct cross-module service calls with an interface, then swap the implementation to an HTTP client (`RestClient`/`WebClient`) or message broker.
3. Move the module's folder and `pom.xml` dependencies into its own repository/service.

---

## HTTP Request Lifecycle

How an incoming HTTP request travels through a Spring Boot application:

```text
Client Request
  │
  ▼
[ Servlet Filters ]        ──► SecurityFilterChain (CORS, JwtAuthFilter, AuthZ)
  │
  ▼
[ DispatcherServlet ]      ──► Front Controller; routes URL to matching handler
  │
  ▼
[ @RestController ]        ──► Deserializes JSON (@RequestBody) & validates (@Valid)
  │
  ▼
[ @Service ]               ──► Executes business rules & manages @Transactional boundary
  │
  ▼
[ @Repository ]            ──► Translates method calls to SQL via Hibernate/JPA
  │
  ▼
[ PostgreSQL DB ]
```

---

## Core Theoretical Concepts

### Inversion of Control (IoC) & Dependency Injection (DI)

- **IoC:** Instead of classes instantiating their own dependencies (`new BookRepository()`), the **Spring IoC Container** (`ApplicationContext`) creates, configures, and manages object lifecycles ("Beans").
- **DI:** Spring injects required beans into dependent classes automatically. Always prefer **Constructor Injection** (`@RequiredArgsConstructor` + `private final`) so dependencies are immutable and easy to mock in unit tests.

### Bean Scopes & Statelessness

- **`singleton` (Default):** Spring creates **one shared instance** of a `@Service`, `@Controller`, or `@Repository` for the entire app. Never store user-specific or request-specific mutable state in class fields (causes race conditions).
- **`prototype`:** Creates a brand-new instance every time the bean is requested.
- **Stateless API Design:** With JWT authentication, the server stores zero session state in memory—every request carries its own credentials, allowing horizontal scaling across multiple containers.

### Entity vs. DTO (Data Transfer Object)

Never expose JPA `@Entity` classes directly in `@RestController` endpoints:

- **Security:** Prevents leaking sensitive fields (e.g., `passwordHash`) or allowing mass-assignment attacks.
- **Decoupling:** Lets you change database column names without breaking external API contracts.
- **Serialization Safety:** Avoids infinite JSON loops and accidental lazy-loading queries on bidirectional relationships.

### ACID & Transaction Boundaries

- **Atomicity:** All database writes inside a `@Transactional` method either commit together or roll back completely if an unchecked exception (`RuntimeException`) occurs.
- **Proxy Mechanism:** Spring wraps `@Transactional` beans in a dynamic proxy. Calling a `@Transactional` method from *another method inside the same class* (`this.saveData()`) bypasses the proxy and ignores the transaction.
