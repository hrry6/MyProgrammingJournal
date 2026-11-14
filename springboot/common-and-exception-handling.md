# Common Utilities & Global Exception Handling (`common/`)

[🏠 Home](../README.md) > [🍃 Spring Boot](./README.md)

---

## `config/` vs `common/` Separation

| Package | Responsibility | Examples |
| :--- | :--- | :--- |
| **`config/`** | Framework wiring, external library beans, security policies | `SecurityConfig`, `JwtService`, `JwtAuthenticationFilter`, `JpaAuditingConfig`, `OpenApiConfig` |
| **`common/`** | Reusable domain-agnostic classes shared across all feature modules | `ApiResponse<T>`, `GlobalExceptionHandler`, `ResourceNotFoundException`, `BaseEntity` |

---

## Unified API Response Wrapper (`ApiResponse.java`)

Provides a consistent JSON envelope across all endpoints:

```java
package com.hrry6.app.common.dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.Builder;
import lombok.Getter;

import java.time.Instant;

@Getter
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiResponse<T> {

    private final boolean success;
    private final String message;
    private final T data;
    private final Object errors;
    @Builder.Default
    private final Instant timestamp = Instant.now();

    public static <T> ApiResponse<T> ok(String message, T data) {
        return ApiResponse.<T>builder()
                .success(true)
                .message(message)
                .data(data)
                .build();
    }

    public static <T> ApiResponse<T> error(String message, Object errors) {
        return ApiResponse.<T>builder()
                .success(false)
                .message(message)
                .errors(errors)
                .build();
    }
}
```

---

## Custom Domain Exceptions

Keep runtime exceptions semantic so services throw clear business errors without knowing HTTP status codes:

```java
package com.hrry6.app.common.exception;

public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

```java
package com.hrry6.app.common.exception;

public class DuplicateResourceException extends RuntimeException {
    public DuplicateResourceException(String message) {
        super(message);
    }
}
```

---

## `GlobalExceptionHandler.java` (`@RestControllerAdvice`)

Centralizes exception-to-HTTP-status translation across all controllers and security filters (via `HandlerExceptionResolver`).

```java
package com.hrry6.app.common.exception;

import com.hrry6.app.common.dto.ApiResponse;
import io.jsonwebtoken.JwtException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    // 400 - DTO @Valid validation failures
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> handleValidationErrors(MethodArgumentNotValidException ex) {
        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
                .forEach(err -> fieldErrors.put(err.getField(), err.getDefaultMessage()));

        return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(ApiResponse.error("Validation failed", fieldErrors));
    }

    // 401 - Wrong login credentials or invalid/expired JWT
    @ExceptionHandler({BadCredentialsException.class, JwtException.class})
    public ResponseEntity<ApiResponse<Void>> handleAuthenticationErrors(RuntimeException ex) {
        return ResponseEntity
                .status(HttpStatus.UNAUTHORIZED)
                .body(ApiResponse.error("Authentication failed: " + ex.getMessage(), null));
    }

    // 403 - Authenticated user lacks required role/permission (@PreAuthorize)
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ApiResponse<Void>> handleAccessDenied(AccessDeniedException ex) {
        return ResponseEntity
                .status(HttpStatus.FORBIDDEN)
                .body(ApiResponse.error("Access denied", null));
    }

    // 404 - Entity not found
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiResponse<Void>> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(ApiResponse.error(ex.getMessage(), null));
    }

    // 409 - Unique constraint / duplicate email or ISBN
    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ApiResponse<Void>> handleDuplicate(DuplicateResourceException ex) {
        return ResponseEntity
                .status(HttpStatus.CONFLICT)
                .body(ApiResponse.error(ex.getMessage(), null));
    }

    // 500 - Catch-all fallback (never leak stack traces to client)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleUnexpected(Exception ex) {
        return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ApiResponse.error("An unexpected internal error occurred", null));
    }
}
```

---

## JPA Auditing (`JpaAuditingConfig` & `BaseEntity`)

Automatically populates `createdAt` and `updatedAt` timestamps on every entity without manual `Instant.now()` assignments.

### Enable Auditing (`config/JpaAuditingConfig.java`)

```java
package com.hrry6.app.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;

@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {
}
```

### Reusable Mapped Superclass (`common/entity/BaseEntity.java`)

```java
package com.hrry6.app.common.entity;

import jakarta.persistence.Column;
import jakarta.persistence.EntityListeners;
import jakarta.persistence.MappedSuperclass;
import lombok.Getter;
import lombok.Setter;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.Instant;

@Getter
@Setter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;
}
```
