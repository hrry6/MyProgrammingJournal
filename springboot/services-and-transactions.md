# Service Layer & Transactions

[🏠 Home](../README.md) > [🍃 Spring Boot](./README.md)

---

## Service Layer Responsibilities

The `@Service` layer sits between HTTP (`@RestController`) and persistence (`@Repository`):

- **Enforces Business Rules:** Checks domain invariants (e.g., duplicate ISBN, available stock, loan limits) and throws semantic domain exceptions (`DuplicateResourceException`, `ResourceNotFoundException`).
- **Owns Transaction Boundaries:** Defines atomic units of work with `@Transactional`.
- **Handles Entity $\leftrightarrow$ DTO Conversion:** Never leaks JPA `@Entity` objects up to the controller.
- **Stays HTTP-Agnostic:** Never imports `HttpServletRequest`, `HttpStatus`, or `ResponseEntity`.

---

## Standard CRUD Service (`BookService.java`)

Apply `@Transactional(readOnly = true)` at the class level as a safe default, then override write methods with `@Transactional`.

```java
package com.hrry6.app.modules.catalog;

import com.hrry6.app.common.exception.DuplicateResourceException;
import com.hrry6.app.common.exception.ResourceNotFoundException;
import com.hrry6.app.modules.catalog.dto.BookResponse;
import com.hrry6.app.modules.catalog.dto.CreateBookRequest;
import com.hrry6.app.modules.catalog.dto.UpdateBookRequest;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.UUID;

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class BookService {

    private final BookRepository bookRepository;

    public Page<BookResponse> getAllBooks(String keyword, Pageable pageable) {
        Page<Book> books = (keyword == null || keyword.isBlank())
                ? bookRepository.findAll(pageable)
                : bookRepository.findByTitleContainingIgnoreCaseOrAuthorContainingIgnoreCase(
                        keyword.trim(), keyword.trim(), pageable
                );

        return books.map(BookResponse::from);
    }

    public BookResponse getBookById(UUID id) {
        Book book = findBookOrThrow(id);
        return BookResponse.from(book);
    }

    @Transactional
    public BookResponse createBook(CreateBookRequest req) {
        if (bookRepository.existsByIsbn(req.isbn())) {
            throw new DuplicateResourceException("Book with ISBN " + req.isbn() + " already exists");
        }

        Book book = Book.builder()
                .title(req.title())
                .author(req.author())
                .isbn(req.isbn())
                .stockAvailable(req.stockAvailable())
                .status(req.stockAvailable() > 0 ? BookStatus.AVAILABLE : BookStatus.OUT_OF_STOCK)
                .build();

        Book saved = bookRepository.save(book);
        return BookResponse.from(saved);
    }

    @Transactional
    public BookResponse updateBook(UUID id, UpdateBookRequest req) {
        Book book = findBookOrThrow(id);

        // Partial update: mutate only non-null fields
        if (req.title() != null) {
            book.setTitle(req.title());
        }
        if (req.author() != null) {
            book.setAuthor(req.author());
        }
        if (req.stockAvailable() != null) {
            book.setStockAvailable(req.stockAvailable());
            book.setStatus(req.stockAvailable() > 0 ? BookStatus.AVAILABLE : BookStatus.OUT_OF_STOCK);
        }

        // No explicit bookRepository.save(book) needed here:
        // Hibernate's Dirty Checking automatically flushes changes on transaction commit.
        return BookResponse.from(book);
    }

    @Transactional
    public void deleteBook(UUID id) {
        Book book = findBookOrThrow(id);
        bookRepository.delete(book);
    }

    // Internal helper shared across service methods
    private Book findBookOrThrow(UUID id) {
        return bookRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Book not found with id: " + id));
    }
}
```

---

## Multi-Entity Transactional Service (`LoanService.java`)

Demonstrates atomic multi-table writes (`loans` + `books`) paired with pessimistic row locking (`findByIdForUpdate`):

```java
package com.hrry6.app.modules.loan;

import com.hrry6.app.common.exception.ResourceNotFoundException;
import com.hrry6.app.modules.auth.User;
import com.hrry6.app.modules.catalog.Book;
import com.hrry6.app.modules.catalog.BookRepository;
import com.hrry6.app.modules.catalog.BookStatus;
import com.hrry6.app.modules.loan.dto.LoanMapper;
import com.hrry6.app.modules.loan.dto.LoanResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDate;
import java.util.UUID;

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class LoanService {

    private final LoanRepository loanRepository;
    private final BookRepository bookRepository;
    private final LoanMapper loanMapper;

    @Transactional
    public LoanResponse borrowBook(UUID bookId, User currentUser) {
        // Lock the book row until this transaction commits to prevent concurrent over-borrowing
        Book book = bookRepository.findByIdForUpdate(bookId)
                .orElseThrow(() -> new ResourceNotFoundException("Book not found with id: " + bookId));

        if (book.getStockAvailable() <= 0) {
            throw new IllegalStateException("Book is currently out of stock");
        }

        // Decrement stock (persisted automatically via dirty checking)
        int updatedStock = book.getStockAvailable() - 1;
        book.setStockAvailable(updatedStock);
        if (updatedStock == 0) {
            book.setStatus(BookStatus.OUT_OF_STOCK);
        }

        Loan loan = Loan.builder()
                .user(currentUser)
                .book(book)
                .dueDate(LocalDate.now().plusDays(14))
                .build();

        Loan savedLoan = loanRepository.save(loan);
        return loanMapper.toResponse(savedLoan);
    }
}
```

---

## `@Transactional` Rules & Gotchas

| Behavior | Rule |
| :--- | :--- |
| **`readOnly = true`** | Skips Hibernate dirty-checking snapshots in memory and hints the JDBC driver/database that no writes will occur (faster queries) |
| **Rollback Trigger** | Automatically rolls back on **unchecked exceptions** (`RuntimeException` & `Error`). Checked exceptions (`IOException`, `Exception`) **commit by default** unless `rollbackFor = Exception.class` is specified |
| **Hibernate Dirty Checking** | Any managed `@Entity` fetched inside a `@Transactional` method is tracked in the persistence context; mutating its fields via setters automatically generates an `UPDATE` SQL statement at commit time |
| **Proxy Self-Invocation** | Calling `this.someTransactionalMethod()` from inside the same class bypasses the Spring AOP proxy and ignores `@Transactional` |
| **Keep Transactions Short** | Never make external HTTP calls, send emails, or run slow file I/O inside a `@Transactional` method—it holds a HikariCP database connection open the entire time |
