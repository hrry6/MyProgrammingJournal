# gRPC in Spring Boot

[🏠 Home](../README.md) > [🍃 Spring Boot](./README.md)

---

## REST vs. gRPC

| Feature | REST (JSON / HTTP 1.1) | gRPC (Protobuf / HTTP 2) |
| :--- | :--- | :--- |
| **Payload Format** | Human-readable text (JSON) | Compact binary (Protocol Buffers) |
| **Transport** | HTTP/1.1 (typically) | HTTP/2 (multiplexing, header compression) |
| **Contract** | Optional (OpenAPI / Swagger) | Strict `.proto` IDL file (code-generated stubs) |
| **Streaming** | Limited (SSE / WebSockets) | Native bidirectional, client, and server streaming |
| **Best Use Case** | Public APIs, browser/mobile clients | Internal microservice-to-microservice calls |

---

## `pom.xml` Setup (Protobuf + `net.devh` Starter)

Add the gRPC server/client starters, `jakarta.annotation-api` (required for Java 21 generated code), and the `protobuf-maven-plugin` to compile `.proto` files during `./mvnw compile`:

```xml
<properties>
    <java.version>21</java.version>
    <grpc.version>1.63.0</grpc.version>
    <protobuf.version>3.25.3</protobuf.version>
    <grpc-starter.version>3.1.0.RELEASE</grpc-starter.version>
</properties>

<dependencies>
    <!-- Use server starter on provider, client starter on consumer (or both) -->
    <dependency>
        <groupId>net.devh</groupId>
        <artifactId>grpc-server-spring-boot-starter</artifactId>
        <version>${grpc-starter.version}</version>
    </dependency>
    <dependency>
        <groupId>net.devh</groupId>
        <artifactId>grpc-client-spring-boot-starter</artifactId>
        <version>${grpc-starter.version}</version>
    </dependency>

    <!-- Required on Java 9+ / 21 for @Generated annotations in protoc output -->
    <dependency>
        <groupId>jakarta.annotation</groupId>
        <artifactId>jakarta.annotation-api</artifactId>
    </dependency>
</dependencies>

<build>
    <extensions>
        <!-- Detects OS/architecture automatically for protoc binary download -->
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.1</version>
        </extension>
    </extensions>

    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:${protobuf.version}:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}</pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

> Run `./mvnw clean compile` after creating or editing `.proto` files so Maven generates the Java message classes and `*Grpc` stubs inside `target/generated-sources/protobuf/`.

---

## Defining the `.proto` Contract

Place `.proto` files inside `src/main/proto/` (in production, share `.proto` files across services via a dedicated Git submodule or internal Maven artifact).

```protobuf
// src/main/proto/book_service.proto
syntax = "proto3";

package catalog;

option java_multiple_files = true;
option java_package = "com.hrry6.grpc.catalog";
option java_outer_classname = "BookServiceProto";

service BookGrpcService {
  // Unary RPC: 1 request -> 1 response
  rpc GetBookStock (BookStockRequest) returns (BookStockResponse);
  rpc DecrementStock (DecrementStockRequest) returns (BookStockResponse);
}

message BookStockRequest {
  string book_id = 1; // Field tags (1, 2, ...) identify binary fields; never reuse deleted numbers
}

message DecrementStockRequest {
  string book_id = 1;
  int32 quantity = 2;
}

message BookStockResponse {
  string book_id = 1;
  string title = 2;
  int32 stock_available = 3;
  bool available = 4;
}
```

### Protobuf Field Rules

- **Field Numbers (`= 1`, `= 2`)**: Encoded into the binary wire format instead of field names. Never change or reuse a field number once deployed.
- **UUIDs & Timestamps**: Pass UUIDs as `string` and timestamps as `int64` (epoch millis) or `google.protobuf.Timestamp`.

---

## Implementing the gRPC Server (`catalog-service`)

### Server Configuration (`application.properties`)

```properties
# HTTP REST port (if hybrid) & dedicated gRPC HTTP/2 port
server.port=8081
grpc.server.port=9090
```

### `@GrpcService` Implementation (`BookGrpcServer.java`)

```java
package com.hrry6.app.modules.catalog.grpc;

import com.hrry6.app.common.exception.ResourceNotFoundException;
import com.hrry6.app.modules.catalog.BookService;
import com.hrry6.app.modules.catalog.dto.BookResponse;
import com.hrry6.grpc.catalog.BookGrpcServiceGrpc;
import com.hrry6.grpc.catalog.BookStockRequest;
import com.hrry6.grpc.catalog.BookStockResponse;
import io.grpc.Status;
import io.grpc.stub.StreamObserver;
import lombok.RequiredArgsConstructor;
import net.devh.boot.grpc.server.service.GrpcService;

import java.util.UUID;

@GrpcService
@RequiredArgsConstructor
public class BookGrpcServer extends BookGrpcServiceGrpc.BookGrpcServiceImplBase {

    private final BookService bookService;

    @Override
    public void getBookStock(BookStockRequest request, StreamObserver<BookStockResponse> responseObserver) {
        try {
            UUID bookId = UUID.fromString(request.getBookId());
            BookResponse book = bookService.getBookById(bookId);

            BookStockResponse response = BookStockResponse.newBuilder()
                    .setBookId(book.id().toString())
                    .setTitle(book.title())
                    .setStockAvailable(book.stockAvailable())
                    .setAvailable(book.stockAvailable() > 0)
                    .build();

            // Send response and close the RPC stream
            responseObserver.onNext(response);
            responseObserver.onCompleted();

        } catch (IllegalArgumentException ex) {
            responseObserver.onError(Status.INVALID_ARGUMENT
                    .withDescription("Invalid UUID format: " + request.getBookId())
                    .asRuntimeException());
        } catch (ResourceNotFoundException ex) {
            responseObserver.onError(Status.NOT_FOUND
                    .withDescription(ex.getMessage())
                    .asRuntimeException());
        }
    }
}
```

---

## Implementing the gRPC Client (`loan-service`)

### Client Configuration (`application.properties`)

```properties
# 'static://' for direct host:port; use 'discovery:///catalog-service' with Eureka/Consul
grpc.client.catalog-service.address=static://localhost:9090
grpc.client.catalog-service.negotiationType=plaintext
```

### `@GrpcClient` Consumer (`CatalogGrpcClient.java`)

```java
package com.hrry6.loan.grpc;

import com.hrry6.common.exception.ResourceNotFoundException;
import com.hrry6.grpc.catalog.BookGrpcServiceGrpc;
import com.hrry6.grpc.catalog.BookStockRequest;
import com.hrry6.grpc.catalog.BookStockResponse;
import io.grpc.Status;
import io.grpc.StatusRuntimeException;
import net.devh.boot.grpc.client.inject.GrpcClient;
import org.springframework.stereotype.Service;

import java.util.UUID;

@Service
public class CatalogGrpcClient {

    @GrpcClient("catalog-service")
    private BookGrpcServiceGrpc.BookGrpcServiceBlockingStub catalogStub;

    public BookStockResponse checkBookStock(UUID bookId) {
        try {
            BookStockRequest request = BookStockRequest.newBuilder()
                    .setBookId(bookId.toString())
                    .build();

            return catalogStub.getBookStock(request);

        } catch (StatusRuntimeException ex) {
            if (ex.getStatus().getCode() == Status.Code.NOT_FOUND) {
                throw new ResourceNotFoundException("Book not found in catalog: " + bookId);
            }
            throw new IllegalStateException("gRPC call to catalog-service failed: " + ex.getStatus(), ex);
        }
    }
}
```

---

## gRPC Status Codes vs. HTTP Equivalents

| gRPC `Status.Code` | HTTP Equivalent | When to Use |
| :--- | :--- | :--- |
| `OK` | `200 OK` | RPC completed successfully |
| `INVALID_ARGUMENT` | `400 Bad Request` | Malformed UUID, negative quantity, invalid field |
| `UNAUTHENTICATED` | `401 Unauthorized` | Missing or expired token in gRPC `Metadata` headers |
| `PERMISSION_DENIED` | `403 Forbidden` | Authenticated caller lacks permission for this RPC |
| `NOT_FOUND` | `404 Not Found` | Requested entity ID does not exist |
| `ALREADY_EXISTS` | `409 Conflict` | Duplicate resource creation |
| `DEADLINE_EXCEEDED` | `504 Gateway Timeout` | Client timeout (`withDeadlineAfter(...)`) expired |
| `UNAVAILABLE` | `503 Service Unavailable` | Target gRPC server is down or unreachable |
