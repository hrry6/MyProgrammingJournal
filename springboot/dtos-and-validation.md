# DTOs & Bean Validation

[🏠 Home](../README.md) > [🍃 Spring Boot](./README.md)

---

## Why Use Java `record` for DTOs?

In Java 21 and Spring Boot 3+, `record` is the standard choice for DTOs:

- **Immutable by default:** Fields are `private final`—request payloads cannot be mutated accidentally mid-flight.
- **Zero boilerplate:** Automatically generates constructor, getters (`req.title()`), `equals()`, `hashCode()`, and `toString()`.
- **Native Jackson & Validation support:** Jackson serializes/deserializes `record` components out of the box, and Jakarta validation annotations work directly on record parameters.

---

## Request & Response DTO Examples

### Request DTO (`CreateBookRequest.java`)

```java
package com.hrry6.app.modules.catalog.dto;

import jakarta.validation.constraints.*;

public record CreateBookRequest(
        @NotBlank(message = "Title is required")
        @Size(max = 200, message = "Title must not exceed 200 characters")
        String title,

        @NotBlank(message = "Author is required")
        @Size(max = 150, message = "Author must not exceed 150 characters")
        String author,

        @NotBlank(message = "ISBN is required")
        @Pattern(regexp = "^(97(8|9))?\\d{9}(\\d|X)\$", message = "Invalid ISBN-10 or ISBN-13 format")
        String isbn,

        @NotNull(message = "Initial stock is required")
        @Min(value = 0, message = "Stock cannot be negative")
        Integer stockAvailable
) {}
```

### Partial Update DTO (`UpdateBookRequest.java`)

Leave fields nullable so clients can send only the fields they want to patch, while still enforcing length/range rules if present:

```java
package com.hrry6.app.modules.catalog.dto;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.Size;

public record UpdateBookRequest(
        @Size(min = 1, max = 200, message = "Title must be between 1 and 200 characters")
        String title,

        @Size(min = 1, max = 150, message = "Author must be between 1 and 150 characters")
        String author,

        @Min(value = 0, message = "Stock cannot be negative")
        Integer stockAvailable
) {}
```

### Response DTO (`BookResponse.java`)

Exposes only safe, client-facing fields—never internal entity state or sensitive columns:

```java
package com.hrry6.app.modules.catalog.dto;

import com.hrry6.app.modules.catalog.Book;
import com.hrry6.app.modules.catalog.BookStatus;

import java.time.Instant;
import java.util.UUID;

public record BookResponse(
        UUID id,
        String title,
        String author,
        String isbn,
        Integer stockAvailable,
        BookStatus status,
        Instant createdAt
) {
    // Static factory method for clean Entity -> DTO conversion
    public static BookResponse from(Book book) {
        return new BookResponse(
                book.getId(),
                book.getTitle(),
                book.getAuthor(),
                book.getIsbn(),
                book.getStockAvailable(),
                book.getStatus(),
                book.getCreatedAt()
        );
    }
}
```

---

## Jakarta Validation Cheatsheet

Trigger validation in controllers by placing `@Valid` before `@RequestBody` (failures automatically route to `MethodArgumentNotValidException` in `GlobalExceptionHandler`):

| Annotation | Supported Types | Validation Rule |
| :--- | :--- | :--- |
| `@NotNull` | Any object | Value must not be `null` (`""` and `"   "` pass) |
| `@NotEmpty` | `String`, `Collection`, `Map`, `Array` | Must not be `null` and length/size must be greater than `0` (`"   "` passes) |
| `@NotBlank` | `String` | Must not be `null` and trimmed length must be greater than `0` (**default for required strings**) |
| `@Size(min, max)` | `String`, `Collection`, `Map`, `Array` | Inclusive length or element count boundaries |
| `@Min(val)` / `@Max(val)` | `Integer`, `Long`, `BigDecimal` | Inclusive numeric lower/upper bounds |
| `@Positive` / `@PositiveOrZero` | Numeric types | Strictly greater than `0` (`> 0`) or `>= 0` |
| `@Email` | `String` | Must match standard email address syntax |
| `@Pattern(regexp = "...")` | `String` | Must match the provided regular expression |
| `@Past` / `@Future` | `LocalDate`, `LocalDateTime`, `Instant` | Temporal comparison against current clock |
| `@Valid` | Nested DTO object / `List<ItemDTO>` | Cascades validation recursively into nested DTO fields |

### Validating Nested Objects & Lists

```java
public record CreateOrderRequest(
        @NotNull(message = "Customer ID is required")
        UUID customerId,

        @NotEmpty(message = "Order must contain at least one item")
        @Valid // Crucial: without @Valid here, constraints inside OrderItemRequest are ignored
        List<OrderItemRequest> items
) {}
```

---

## 4. Entity <-> DTO Mapping Patterns

### Option A: Static Factory & Builder (Recommended for Most Projects)

Explicit, compile-time safe, zero magic, and easy to trace with IDE navigation:

```java
// Request DTO -> Entity (inside Service layer)
Book book = Book.builder()
        .title(req.title())
        .author(req.author())
        .isbn(req.isbn())
        .stockAvailable(req.stockAvailable())
        .status(req.stockAvailable() > 0 ? BookStatus.AVAILABLE : BookStatus.OUT_OF_STOCK)
        .build();

// Entity -> Response DTO
BookResponse response = BookResponse.from(savedBook);
```

### Option B: Dedicated Mapper Component (When Mapping Involves Multiple Entities)

Extract a `@Component` mapper when constructing a DTO requires combining data from multiple entities or formatting logic:

```java
package com.hrry6.app.modules.loan.dto;

import com.hrry6.app.modules.loan.Loan;
import org.springframework.stereotype.Component;

@Component
public class LoanMapper {

    public LoanResponse toResponse(Loan loan) {
        return new LoanResponse(
                loan.getId(),
                loan.getUser().getId(),
                loan.getUser().getFullName(),
                loan.getBook().getId(),
                loan.getBook().getTitle(),
                loan.getDueDate(),
                loan.getReturnedAt()
        );
    }
}
```
