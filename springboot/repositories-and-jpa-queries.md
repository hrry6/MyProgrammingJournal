
# Repositories & JPA Queries

[🏠 Home](../README.md) > [🍃 Spring Boot](./README.md)

---

## Core `JpaRepository` & Derived Queries

Spring Data JPA generates SQL implementations automatically at startup from interface method names. Always annotate `@Repository` only when writing custom DAO implementations (`JpaRepository` interfaces are already registered as beans automatically).

```java
package com.hrry6.app.modules.catalog;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

public interface BookRepository extends JpaRepository<Book, UUID> {

    // Exact match & existence checks
    Optional<Book> findByIsbn(String isbn);
    boolean existsByIsbn(String isbn);

    // Case-insensitive search with pagination
    Page<Book> findByTitleContainingIgnoreCaseOrAuthorContainingIgnoreCase(
            String title,
            String author,
            Pageable pageable
    );

    // Filtering by enum & numeric comparison
    List<Book> findByStatusAndStockAvailableGreaterThan(BookStatus status, Integer minStock);
}
```

### Common Derived Query Keywords

| Keyword | Example Method | Generated SQL Condition |
| :--- | :--- | :--- |
| `Is` / `Equals` | `findByIsbn(String isbn)` | `WHERE isbn = ?` |
| `IgnoreCase` | `findByEmailIgnoreCase(String email)` | `WHERE LOWER(email) = LOWER(?)` |
| `Containing` | `findByTitleContaining(String kw)` | `WHERE title LIKE '%kw%'` |
| `StartingWith` | `findByIsbnStartingWith(String prefix)` | `WHERE isbn LIKE 'prefix%'` |
| `GreaterThan` / `LessThan` | `findByStockAvailableGreaterThan(int n)` | `WHERE stock_available > ?` |
| `Between` | `findByDueDateBetween(LocalDate s, LocalDate e)` | `WHERE due_date BETWEEN ? AND ?` |
| `In` | `findByStatusIn(Collection<BookStatus> statuses)` | `WHERE status IN (?, ?)` |
| `IsNull` / `IsNotNull` | `findByReturnedAtIsNull()` | `WHERE returned_at IS NULL` |
| `OrderBy...Desc` | `findByUserIdOrderByCreatedAtDesc(UUID userId)` | `WHERE user_id = ? ORDER BY created_at DESC` |

---

## `Page<T>` vs `Slice<T>` (Pagination)

| Return Type | Extra `COUNT(*)` Query? | Best Used For |
| :--- | :--- | :--- |
| **`Page<T>`** | **Yes** (executes a second `SELECT count(...)` query) | Traditional numbered pagination (`totalElements`, `totalPages`) |
| **`Slice<T>`** | **No** (fetches `pageSize + 1` rows to check `hasNext()`) | Infinite scroll or "Load More" buttons on large tables |

---

## Custom Queries (`@Query` & `@Modifying`)

Use `@Query` when method names get too long or when you need PostgreSQL-specific SQL features.

```java
package com.hrry6.app.modules.loan;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.LocalDate;
import java.util.List;
import java.util.UUID;

public interface LoanRepository extends JpaRepository<Loan, UUID> {

    // 1. JPQL Query (references Java Entity & field names, NOT table/column names)
    @Query("""
            SELECT l FROM Loan l
            WHERE l.returnedAt IS NULL
              AND l.dueDate < :today
            """)
    List<Loan> findOverdueLoans(@Param("today") LocalDate today);

    // 2. Native PostgreSQL Query (references actual SQL tables & PostgreSQL syntax)
    @Query(
        value = """
                SELECT * FROM loans
                WHERE user_id = :userId
                  AND returned_at IS NULL
                ORDER BY due_date ASC
                """,
        nativeQuery = true
    )
    List<Loan> findActiveLoansByUserNative(@Param("userId") UUID userId);

    // 3. Bulk UPDATE / DELETE (must be called inside a @Transactional service method)
    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("UPDATE Loan l SET l.returnedAt = :returnDate WHERE l.id = :loanId AND l.returnedAt IS NULL")
    int markAsReturned(@Param("loanId") UUID loanId, @Param("returnDate") LocalDate returnDate);
}
```

> **Why `@Modifying(clearAutomatically = true)` matters:** Bulk JPQL `UPDATE`/`DELETE` queries bypass Hibernate's 1st-level persistence context cache. Setting `clearAutomatically = true` evicts stale cached entities so subsequent `findById()` calls in the same transaction read fresh data from PostgreSQL.

---

## The N+1 Query Problem

### What Is the N+1 Problem?

The N+1 problem happens when fetching a list of `N` parent entities triggers **1 initial query** for the parents, followed by **N additional queries** to load a `LAZY` relationship on each item inside a loop or DTO mapper.

```java
// Suppose there are 50 active loans in the database:
List<Loan> loans = loanRepository.findByReturnedAtIsNull(); // Query #1: SELECT * FROM loans WHERE returned_at IS NULL

// Mapping each Loan -> LoanResponse touches lazy associations (loan.getUser() & loan.getBook()):
List<LoanResponse> responses = loans.stream()
        .map(loan -> new LoanResponse(
                loan.getId(),
                loan.getUser().getFullName(), // Triggers Query #2, #4, #6... (50 queries to users)
                loan.getBook().getTitle()     // Triggers Query #3, #5, #7... (50 queries to books)
        ))
        .toList();
// Result: 1 + 50 + 50 = 101 SQL queries for a single API call!
```

---

### Ways to Solve the N+1 Problem

#### Solution 1: `@EntityGraph` (Cleanest for Derived Queries & Pagination)

Instructs Hibernate to fetch specified associations in a single `LEFT OUTER JOIN` query without writing custom JPQL.

```java
public interface LoanRepository extends JpaRepository<Loan, UUID> {

    @EntityGraph(attributePaths = {"user", "book"})
    List<Loan> findByReturnedAtIsNull();

    @EntityGraph(attributePaths = {"user", "book"})
    Page<Loan> findByUserId(UUID userId, Pageable pageable);
}
```

#### Solution 2: Explicit JPQL `JOIN FETCH`

Gives full control over `JOIN FETCH` (inner join) vs `LEFT JOIN FETCH` (outer join) in custom queries.

```java
@Query("""
        SELECT l FROM Loan l
        JOIN FETCH l.user
        JOIN FETCH l.book
        WHERE l.returnedAt IS NULL
        """)
List<Loan> findActiveLoansWithDetails();
```

> **Pagination Gotcha (`HHH90003004`):** Never combine `JOIN FETCH` with `Pageable` on a **`@OneToMany` collection**—Hibernate will fetch the entire table into JVM memory and paginate in-memory (`firstResult/maxResults specified with collection fetch; applying in memory!`).
>
> - For `@ManyToOne` / `@OneToOne` (like `Loan -> User`, `Loan -> Book`), `JOIN FETCH` / `@EntityGraph` with `Pageable` is safe because row count does not multiply.
> - For `@OneToMany` collections with pagination, use `@BatchSize` (Solution 4) or fetch paginated parent IDs first, then `JOIN FETCH` by `WHERE p.id IN :ids`.

#### Solution 3: Constructor DTO Projections (Best for Read-Heavy Endpoints)

Selects only the exact columns needed directly into a Java `record`, bypassing entity management and lazy proxies completely.

```java
@Query("""
        SELECT new com.hrry6.app.modules.loan.dto.LoanSummaryDto(
            l.id,
            u.fullName,
            b.title,
            l.dueDate
        )
        FROM Loan l
        JOIN l.user u
        JOIN l.book b
        WHERE l.returnedAt IS NULL
        """)
List<LoanSummaryDto> findAllActiveLoanSummaries();
```

#### Solution 4: Hibernate Batch Fetching (`@BatchSize`)

Groups lazy-loading queries into `WHERE id IN (?, ?, ...)` batches instead of firing one query per entity. Useful as a global safety net in `application.properties`:

```properties
# Turns 1 + 50 queries into 1 + 2 queries (batches of 25 IDs per IN clause)
spring.jpa.properties.hibernate.default_batch_fetch_size=25
```

---

## Pessimistic Locking for Concurrency (`@Lock`)

When two requests try to borrow the last copy of a book simultaneously, standard reads cause a race condition. Use `PESSIMISTIC_WRITE` (`SELECT ... FOR UPDATE` in PostgreSQL) inside a `@Transactional` service method to lock the row until the transaction commits:

```java
public interface BookRepository extends JpaRepository<Book, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT b FROM Book b WHERE b.id = :id")
    Optional<Book> findByIdForUpdate(@Param("id") UUID id);
}
```

### Summary: Choosing the Right Query Strategy

| Scenario | Recommended Approach |
| :--- | :--- |
| Simple single-table lookup | Derived method (`findByIsbn`, `existsByEmail`) |
| Loading parent with `@ManyToOne` associations | `@EntityGraph(attributePaths = {...})` or `JOIN FETCH` |
| Paginating parent with `@OneToMany` collections | `default_batch_fetch_size=25` or 2-step ID fetch |
| Dashboard / reporting list with flat fields | JPQL Constructor DTO Projection (`SELECT new ...`) |
| Updating stock / balances under concurrent traffic | `@Lock(LockModeType.PESSIMISTIC_WRITE)` inside `@Transactional` |
