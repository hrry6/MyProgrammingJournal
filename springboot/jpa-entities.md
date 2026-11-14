
# JPA Entities & Relationships

[🏠 Home](../README.md) > [🍃 Spring Boot](./README.md)

---

## Core Entity Anatomy (`Book.java`)

Extends `BaseEntity` (from `common/entity/`) to inherit automatic `created_at` and `updated_at` auditing columns.

```java
package com.hrry6.app.modules.catalog;

import com.hrry6.app.common.entity.BaseEntity;
import jakarta.persistence.*;
import lombok.*;

import java.util.UUID;

@Entity
@Table(
    name = "books",
    indexes = {
        @Index(name = "idx_books_isbn", columnList = "isbn", unique = true)
    }
)
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Book extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false, length = 200)
    private String title;

    @Column(nullable = false, length = 150)
    private String author;

    @Column(nullable = false, unique = true, length = 20)
    private String isbn;

    @Column(name = "stock_available", nullable = false)
    private Integer stockAvailable;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private BookStatus status;
}
```

### Key Mapping Annotations

| Annotation | Purpose | Notes |
| :--- | :--- | :--- |
| `@Entity` | Marks class as a JPA persistence entity | Requires a `protected` or `public` no-arg constructor (`@NoArgsConstructor`) |
| `@Table(name = "...")` | Maps entity to an explicit SQL table name | Always use plural `snake_case` (e.g., `books`, `users`) to avoid SQL reserved keyword collisions (`user`, `order`) |
| `@Id` + `@GeneratedValue` | Primary key definition | `GenerationType.UUID` generates standard v4 UUIDs natively in Spring Boot 3+ / Hibernate 6 |
| `@Column(...)` | Column constraints & naming | Hibernate auto-converts `camelCase` fields to `snake_case` columns, or specify `name = "..."` explicitly |
| `@Enumerated(EnumType.STRING)` | Persists enum as readable text (`"AVAILABLE"`) | **Never** use `EnumType.ORDINAL`—reordering enum constants corrupts existing database rows |

---

## Entity Relationships (`Loan.java`)

```java
package com.hrry6.app.modules.loan;

import com.hrry6.app.common.entity.BaseEntity;
import com.hrry6.app.modules.auth.User;
import com.hrry6.app.modules.catalog.Book;
import jakarta.persistence.*;
import lombok.*;

import java.time.LocalDate;
import java.util.UUID;

@Entity
@Table(name = "loans")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Loan extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    // Owning side of the relationship (holds the foreign key column user_id)
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "book_id", nullable = false)
    private Book book;

    @Column(name = "due_date", nullable = false)
    private LocalDate dueDate;

    @Column(name = "returned_at")
    private LocalDate returnedAt;
}
```

### Relationship Rules & Fetch Types

| Annotation | Default Fetch | Recommended Fetch | Foreign Key Placement |
| :--- | :--- | :--- | :--- |
| `@ManyToOne` | `EAGER` | **`FetchType.LAZY`** | Owning side (`@JoinColumn(name = "user_id")`) |
| `@OneToOne` | `EAGER` | **`FetchType.LAZY`** | Owning side (`@JoinColumn(name = "profile_id")`) |
| `@OneToMany` | `LAZY` | `FetchType.LAZY` | Inverse side (`mappedBy = "user"`) |
| `@ManyToMany` | `LAZY` | `FetchType.LAZY` | Join table (`@JoinTable`), or model an explicit join entity |

---

## Security User Entity (`User.java` + `UserDetails`)

Implementing Spring Security's `UserDetails` directly on the `User` entity allows `UserRepository.findByEmail()` to feed straight into `JwtService` and `DaoAuthenticationProvider`.

```java
package com.hrry6.app.modules.auth;

import com.hrry6.app.common.entity.BaseEntity;
import jakarta.persistence.*;
import lombok.*;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.List;
import java.util.UUID;

@Entity
@Table(name = "users")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User extends BaseEntity implements UserDetails {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false, length = 100)
    private String fullName;

    @Column(nullable = false, unique = true, length = 150)
    private String email;

    @Column(nullable = false)
    private String password;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private Role role; // e.g., ROLE_MEMBER, ROLE_ADMIN

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority(role.name()));
    }

    @Override
    public String getUsername() {
        return email; // Use email as the unique login identifier
    }

    @Override
    public boolean isAccountNonExpired() { return true; }

    @Override
    public boolean isAccountNonLocked() { return true; }

    @Override
    public boolean isCredentialsNonExpired() { return true; }

    @Override
    public boolean isEnabled() { return true; }
}
```

---

## Critical JPA & Lombok Gotchas

1. **Never put `@Data` on `@Entity` classes:**
   - Lombok's `@Data` generates `equals()`, `hashCode()`, and `toString()` using **all fields**.
   - Touching a `LAZY` relationship inside `toString()` or `hashCode()` triggers unexpected SQL queries, `LazyInitializationException` outside a transaction, or `StackOverflowError` on bidirectional links.
   - **Solution:** Always use `@Getter`, `@Setter`, `@NoArgsConstructor`, `@AllArgsConstructor`, and `@Builder`.
2. **Always override default `EAGER` on `@ManyToOne` / `@OneToOne`:**
   - Leaving `@ManyToOne` without `fetch = FetchType.LAZY` forces Hibernate to fetch parent entities on every single query even when you only need the foreign key.
3. **Avoid `CascadeType.REMOVE` on `@ManyToOne`:**
   - Deleting a child record (`Loan`) should never cascade-delete the parent (`User` or `Book`).
