# Controllers & REST API Design

[🏠 Home](../README.md) > [🍃 Spring Boot](./README.md)

---

## Controller Layer Responsibilities

A controller should stay thin and act strictly as an HTTP adapter:

1. Map HTTP verbs and URL routes (`@GetMapping`, `@PostMapping`, etc.).
2. Extract and validate inputs (`@Valid @RequestBody`, `@PathVariable`, `@RequestParam`).
3. Enforce endpoint-level permissions (`@PreAuthorize`).
4. Delegate business execution to `@Service` and wrap the result in `ResponseEntity<ApiResponse<T>>`.

---

## REST Controller Example (`BookController.java`)

```java
package com.hrry6.app.modules.catalog;

import com.hrry6.app.common.dto.ApiResponse;
import com.hrry6.app.modules.catalog.dto.BookResponse;
import com.hrry6.app.modules.catalog.dto.CreateBookRequest;
import com.hrry6.app.modules.catalog.dto.UpdateBookRequest;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@RestController
@RequestMapping("/api/v1/books")
@RequiredArgsConstructor
public class BookController {

    private final BookService bookService;

    // GET /api/v1/books?keyword=clean&page=0&size=10&sort=title,asc
    @GetMapping
    public ResponseEntity<ApiResponse<Page<BookResponse>>> getAllBooks(
            @RequestParam(required = false) String keyword,
            @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC) Pageable pageable
    ) {
        Page<BookResponse> books = bookService.getAllBooks(keyword, pageable);
        return ResponseEntity.ok(ApiResponse.ok("Books retrieved successfully", books));
    }

    // GET /api/v1/books/{id}
    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<BookResponse>> getBookById(@PathVariable UUID id) {
        BookResponse book = bookService.getBookById(id);
        return ResponseEntity.ok(ApiResponse.ok("Book retrieved successfully", book));
    }

    // POST /api/v1/books (Admin only)
    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ApiResponse<BookResponse>> createBook(@Valid @RequestBody CreateBookRequest req) {
        BookResponse created = bookService.createBook(req);
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.ok("Book created successfully", created));
    }

    // PATCH /api/v1/books/{id} (Admin only)
    @PatchMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ApiResponse<BookResponse>> updateBook(
            @PathVariable UUID id,
            @Valid @RequestBody UpdateBookRequest req
    ) {
        BookResponse updated = bookService.updateBook(id, req);
        return ResponseEntity.ok(ApiResponse.ok("Book updated successfully", updated));
    }

    // DELETE /api/v1/books/{id} (Admin only)
    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteBook(@PathVariable UUID id) {
        bookService.deleteBook(id);
        return ResponseEntity.noContent().build();
    }
}
```

---

## Accessing the Logged-In User (`@AuthenticationPrincipal`)

Because `User` implements `UserDetails` and `JwtAuthenticationFilter` sets it as the principal in `SecurityContextHolder`, inject it directly into controller methods using `@AuthenticationPrincipal`:

```java
package com.hrry6.app.modules.loan;

import com.hrry6.app.common.dto.ApiResponse;
import com.hrry6.app.modules.auth.User;
import com.hrry6.app.modules.loan.dto.LoanResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@RestController
@RequestMapping("/api/v1/loans")
@RequiredArgsConstructor
public class LoanController {

    private final LoanService loanService;

    @PostMapping("/borrow/{bookId}")
    public ResponseEntity<ApiResponse<LoanResponse>> borrowBook(
            @PathVariable UUID bookId,
            @AuthenticationPrincipal User currentUser
    ) {
        LoanResponse loan = loanService.borrowBook(bookId, currentUser);
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.ok("Book borrowed successfully", loan));
    }
}
```

---

## Parameter Binding & HTTP Status Cheatsheet

### Request Parameter Annotations

| Annotation | Example URL / Payload | Usage |
| :--- | :--- | :--- |
| `@PathVariable` | `/api/v1/books/{id}` | Extracts dynamic URI segments (`UUID`, `Long`, `String`) |
| `@RequestParam` | `/api/v1/books?keyword=java&page=0` | Extracts query string parameters (`required = false`, `defaultValue = "..."`) |
| `@RequestBody` | `{"title": "Clean Code"}` | Deserializes JSON request body into a DTO via Jackson |
| `@Valid` | `@Valid @RequestBody CreateBookRequest` | Triggers Jakarta Bean Validation before the controller method runs |
| `@PageableDefault` | `?page=0&size=20&sort=title,asc` | Automatically binds pagination & sorting query params into Spring's `Pageable` |
| `@RequestHeader` | `X-Request-ID: abc-123` | Reads custom HTTP headers |
| `@AuthenticationPrincipal` | Injected from `SecurityContext` | Retrieves the currently authenticated `UserDetails` instance |

### Standard HTTP Status Codes

| Status Code | Constant | When to Return |
| :--- | :--- | :--- |
| **`200 OK`** | `HttpStatus.OK` | Successful `GET`, `PUT`, `PATCH`, or login action returning a response body |
| **`201 Created`** | `HttpStatus.CREATED` | Successful `POST` that creates a new resource |
| **`204 No Content`** | `HttpStatus.NO_CONTENT` | Successful `DELETE` (or action) with an empty response body |
| **`400 Bad Request`** | `HttpStatus.BAD_REQUEST` | Malformed JSON or `@Valid` DTO validation failure |
| **`401 Unauthorized`** | `HttpStatus.UNAUTHORIZED` | Missing, expired, or invalid JWT / wrong login credentials |
| **`403 Forbidden`** | `HttpStatus.FORBIDDEN` | Valid JWT, but user lacks the required role (`@PreAuthorize`) |
| **`404 Not Found`** | `HttpStatus.NOT_FOUND` | Requested resource ID does not exist in the database |
| **`409 Conflict`** | `HttpStatus.CONFLICT` | Duplicate unique field (e.g., email or ISBN already registered) |

### `@PreAuthorize` Expression Quick Reference

- `@PreAuthorize("hasRole('ADMIN')")` — Checks for authority `"ROLE_ADMIN"` (Spring automatically prepends `ROLE_`).
- `@PreAuthorize("hasAnyRole('ADMIN', 'LIBRARIAN')")` — Allows any of the listed roles.
- `@PreAuthorize("#userId == authentication.principal.id or hasRole('ADMIN')")` — Resource-owner check combined with admin override.
