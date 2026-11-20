# Microservice Project & Folder Structure

[🏠 Home](../README.md) > [🍃 Spring Boot](./README.md)

---

## Repository Layout (Multi-Service Workspace)

In a **Monorepo** (recommended for solo developers and small-to-medium teams), each microservice is an independent Maven project sitting under one root repository, alongside a shared contract module for `.proto` definitions:

```text
library-microservices/
├── docker-compose.yml              # Spins up Postgres instances, RabbitMQ/Kafka, Redis, etc.
├── proto-contracts/                # Shared gRPC .proto definitions (compiled into a JAR)
│   ├── pom.xml
│   └── src/main/proto/
│       ├── catalog_service.proto
│       └── user_service.proto
├── api-gateway/                    # Spring Cloud Gateway (Routing, Rate Limiting, JWT Auth)
│   ├── pom.xml
│   └── src/...
├── auth-service/                   # User registration, login, JWT issuing
│   ├── pom.xml
│   └── src/...
├── catalog-service/                # Book inventory & stock management
│   ├── pom.xml
│   └── src/...
└── loan-service/                   # Borrowing & return transactions
    ├── pom.xml
    └── src/...
```

> **Polyrepo Alternative:** Each folder (`catalog-service`, `loan-service`, `proto-contracts`) lives in its own Git repository, and `proto-contracts` is published as a versioned Maven artifact (`com.hrry6:proto-contracts:1.0.0`).

---

## Inside a Single Microservice (`loan-service`)

Each microservice is a complete Spring Boot application. Classic layers (`controller`, `service`, `repository`, `entity`, `dto`) remain the core, extended with folders for inbound/outbound inter-service communication:

```text
loan-service/
├── pom.xml
├── Dockerfile
└── src
    ├── main
    │   ├── java/com/hrry6/loan
    │   │   ├── LoanServiceApplication.java
    │   │   ├── config/                     # Security (header/JWT filter), gRPC/RestClient beans, Kafka config
    │   │   │   ├── HttpClientConfig.java
    │   │   │   └── SecurityConfig.java
    │   │   ├── common/                     # ApiResponse<T>, GlobalExceptionHandler, BaseEntity
    │   │   │   ├── dto/ApiResponse.java
    │   │   │   ├── entity/BaseEntity.java
    │   │   │   └── exception/GlobalExceptionHandler.java
    │   │   ├── controller/                 # Inbound REST endpoints for external clients / Gateway
    │   │   │   └── LoanController.java
    │   │   ├── service/                    # Core domain business logic & @Transactional boundaries
    │   │   │   └── LoanService.java
    │   │   ├── repository/                 # Spring Data JPA interfaces (connects ONLY to loan_db)
    │   │   │   └── LoanRepository.java
    │   │   ├── entity/                     # JPA @Entity classes owned exclusively by this service
    │   │   │   └── Loan.java
    │   │   ├── dto/                        # Request/Response records & mappers
    │   │   │   ├── BorrowBookRequest.java
    │   │   │   └── LoanResponse.java
    │   │   ├── grpc/                       # Inbound (@GrpcService) & Outbound (@GrpcClient) RPC adapters
    │   │   │   ├── CatalogGrpcClient.java
    │   │   │   └── LoanGrpcServer.java
    │   │   ├── client/                     # Outbound declarative HTTP clients (@HttpExchange / RestClient)
    │   │   │   └── CatalogHttpClient.java
    │   │   └── messaging/                  # Async event publishers & listeners (Kafka / RabbitMQ)
    │   │       ├── LoanEventPublisher.java
    │   │       └── dto/BookBorrowedEvent.java
    │   └── resources/
    │       └── application.properties
    └── test
        ├── java/com/hrry6/loan
        │   ├── AbstractIntegrationTest.java
        │   ├── service/LoanServiceTest.java
        │   └── controller/LoanIntegrationTest.java
        └── resources/
            └── application-test.properties
```

---

## What Stays vs. What Changes from a Monolith

| Layer / Folder | Still Exists? | How It Behaves in a Microservice |
| :--- | :--- | :--- |
| **`controller/`** | **Yes** | Exposes REST endpoints for frontend/mobile clients (usually routed through API Gateway) |
| **`service/`** | **Yes** | Orchestrates local DB writes + calls `grpc/` or `client/` adapters to fetch external data |
| **`repository/`** | **Yes** | Queries **only this service's own database** (`loan_db`); never accesses another service's tables |
| **`entity/`** | **Yes** | Contains **only local domain entities** (`Loan`); foreign entities are stored as scalar IDs (`UUID bookId`, `UUID userId`) |
| **`dto/`** | **Yes** | Holds REST request/response `record`s plus external response models returned by `client/` |
| **`grpc/` & `client/`** | **New** | Replaces direct cross-module Java service calls with network calls (`@GrpcClient` or `@HttpExchange`) |
| **`messaging/`** | **New** | Replaces in-memory Spring `ApplicationEvent`s with message broker producers/consumers |

---

## Handling Shared `.proto` Files Cleanly

Avoid copy-pasting `.proto` files manually between services—they quickly drift out of sync. Use one of two approaches:

1. **Dedicated `proto-contracts` Maven Module (Recommended):**
   - Put all `.proto` files and `protobuf-maven-plugin` inside `proto-contracts/pom.xml`.
   - Run `./mvnw install` (or publish to GitHub Packages / Nexus).
   - Both `catalog-service` (server) and `loan-service` (client) simply import `<artifactId>proto-contracts</artifactId>` in their `pom.xml`.
2. **Git Submodule:**
   - Store `.proto` files in a standalone `library-protos` Git repository and mount it into `src/main/proto` inside each microservice.

---

## Practical Rule: Package-by-Layer vs. Package-by-Feature Inside a Microservice

- Because a microservice already represents **one bounded business domain** (e.g., `loan-service`), flat **package-by-layer** (`controller/`, `service/`, `repository/`, `entity/`, `dto/`) is clean and sufficient.
- If a single microservice grows to manage 4–5 closely related sub-domains, group them using **package-by-feature** (`modules/borrowing/`, `modules/fines/`, `modules/reservations/`) just like a Modular Monolith.
