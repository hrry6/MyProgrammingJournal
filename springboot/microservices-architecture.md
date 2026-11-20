# Microservices Architecture & Inter-Service Communication

[🏠 Home](../README.md) > [🍃 Spring Boot](./README.md)

---

## Microservice Ecosystem Building Blocks

```text
                       [ Client / Frontend ]
                                 │
                                 ▼
                  [ Spring Cloud API Gateway ]
                  (Routing, Rate Limit, JWT Auth)
                     │                      │
         ┌───────────┘                      └───────────┐
         ▼ (REST / gRPC)                                ▼ (REST / gRPC)
[ Catalog Service ] ───(Async Events: Kafka/RMQ)───► [ Loan Service ]
         │                                              │
         ▼                                              ▼
  (Catalog DB)                                     (Loan DB)
```

| Component | Role | Typical Spring Stack |
| :--- | :--- | :--- |
| **API Gateway** | Single entry point; handles routing, TLS termination, rate limiting, and JWT validation | Spring Cloud Gateway |
| **Service Discovery** | Dynamic registry mapping service names (`catalog-service`) to live container IPs/ports | Netflix Eureka, Consul, or Kubernetes DNS |
| **Centralized Config** | Externalizes environment properties across all services without rebuilding JARs | Spring Cloud Config Server or K8s ConfigMaps |
| **Sync Communication** | Direct request/response calls when immediate data is required | `RestClient`, `@HttpExchange`, or **gRPC** |
| **Async Communication** | Event-driven decoupling for side effects (notifications, audit logs, eventual consistency) | Spring Cloud Stream, Apache Kafka, RabbitMQ |
| **Observability** | Distributed request tracing (`traceId`, `spanId`) and metrics across service hops | Micrometer Tracing + Zipkin/Tempo + Prometheus |

---

## Synchronous Communication (`RestClient` & `@HttpExchange`)

In Spring Boot 3.2+, `RestClient` replaces legacy `RestTemplate` as the standard synchronous HTTP client, while `@HttpExchange` provides declarative interface-based clients natively (without needing third-party OpenFeign).

### A. Declarative HTTP Interface (`CatalogClient.java`)

```java
package com.hrry6.loan.client;

import com.hrry6.loan.client.dto.BookSummaryResponse;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.service.annotation.GetExchange;
import org.springframework.web.service.annotation.HttpExchange;
import org.springframework.web.service.annotation.PostExchange;

import java.util.UUID;

@HttpExchange("/api/v1/books")
public interface CatalogClient {

    @GetExchange("/{id}")
    BookSummaryResponse getBookById(@PathVariable UUID id);

    @PostExchange("/{id}/reserve")
    void reserveStock(@PathVariable UUID id);
}
```

### B. Client Configuration Bean (`HttpClientConfig.java`)

```java
package com.hrry6.loan.config;

import com.hrry6.loan.client.CatalogClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpStatusCode;
import org.springframework.web.client.RestClient;
import org.springframework.web.client.support.RestClientAdapter;
import org.springframework.web.service.invoker.HttpServiceProxyFactory;

@Configuration
public class HttpClientConfig {

    @Bean
    public CatalogClient catalogClient(@Value("${services.catalog.url}") String catalogBaseUrl) {
        RestClient restClient = RestClient.builder()
                .baseUrl(catalogBaseUrl)
                .defaultStatusHandler(HttpStatusCode::is4xxClientError, (req, res) -> {
                    throw new IllegalArgumentException("Catalog client error: " + res.getStatusCode());
                })
                .build();

        RestClientAdapter adapter = RestClientAdapter.create(restClient);
        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(adapter).build();
        return factory.createClient(CatalogClient.class);
    }
}
```

---

## Fault Tolerance with Resilience4j

If `catalog-service` slows down or crashes, cascading timeouts can exhaust Tomcat threads in `loan-service`. Wrap remote calls with a **Circuit Breaker** and **Fallback**.

### `pom.xml` Dependency

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

### `application.properties`

```properties
resilience4j.circuitbreaker.instances.catalogService.slidingWindowSize=10
resilience4j.circuitbreaker.instances.catalogService.failureRateThreshold=50
resilience4j.circuitbreaker.instances.catalogService.waitDurationInOpenState=10s
resilience4j.retry.instances.catalogService.maxAttempts=3
resilience4j.retry.instances.catalogService.waitDuration=500ms
```

### Service Usage with Fallback

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class CatalogIntegrationService {

    private final CatalogClient catalogClient;

    @CircuitBreaker(name = "catalogService", fallbackMethod = "fallbackGetBook")
    @Retry(name = "catalogService")
    public BookSummaryResponse fetchBook(UUID bookId) {
        return catalogClient.getBookById(bookId);
    }

    // Fallback signature must match original parameters + Throwable
    private BookSummaryResponse fallbackGetBook(UUID bookId, Throwable ex) {
        log.warn("Catalog service unavailable for bookId={}: {}", bookId, ex.getMessage());
        return new BookSummaryResponse(bookId, "Unavailable (Cached/Fallback)", 0);
    }
}
```

---

## Distributed Data & Consistency Rules

### A. Database-per-Service Pattern

- **Strict Isolation:** Each microservice owns its own PostgreSQL database/schema. Never query another service's tables directly or share JPA `@Entity` classes across services.
- **Reference by ID:** Store `UUID bookId` inside `Loan` instead of a JPA `@ManyToOne Book` entity.

### B. Transactional Outbox Pattern (Reliable Events)

Updating a local database and publishing a Kafka/RabbitMQ event in the same method is not atomic—if the broker fails after DB commit, the event is lost.

1. Save the business entity (`Loan`) **and** an `OutboxEvent` row inside the **same local `@Transactional` DB transaction**.
2. A background worker (`@Scheduled` or Debezium CDC) polls unpublished `OutboxEvent` rows, publishes them to the message broker, and marks them `PROCESSED`.

---

## Microservice Security Pattern (Gateway Offloading)

```text
Client (Bearer JWT) ──► [ API Gateway ] ──(Validates JWT, extracts userId & roles)──► Downstream Service
                                                Adds headers:
                                                X-User-Id: 550e8400-...
                                                X-User-Roles: ROLE_MEMBER
```

1. **API Gateway** verifies the JWT signature and expiration once at the edge.
2. Downstream services trust internal network traffic (secured via private VPC/mesh/mTLS) and read `X-User-Id` and `X-User-Roles` headers in a lightweight filter to populate `SecurityContextHolder`—avoiding repeated database user lookups on every internal hop.
