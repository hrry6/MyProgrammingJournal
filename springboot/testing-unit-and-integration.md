
# Unit & Integration Testing

[🏠 Home](../README.md) > [🍃 Spring Boot](./README.md)

---

## Testing Strategy Overview

| Test Type | Annotations | Spring Context? | Database | Speed | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Unit Test** | `@ExtendWith(MockitoExtension.class)` | **No** | Mocked (`@Mock`) | Very Fast (ms) | Verify business rules, branching, and exception throwing in `@Service` |
| **Integration Test** | `@SpringBootTest`, `@AutoConfigureMockMvc`, `@Testcontainers` | **Yes** (Full) | Real PostgreSQL (Docker) | Slower (sec) | Verify HTTP routing, `@Valid`, Spring Security RBAC, SQL schema, and DB queries end-to-end |

---

## Service Unit Test (`BookServiceTest.java`)

Tests business logic in pure Java without booting Spring or Docker. Uses **BDDMockito** (`given`/`willReturn`) and **AssertJ** (`assertThat`, `assertThatThrownBy`).

```java
package com.hrry6.app.modules.catalog;

import com.hrry6.app.common.exception.DuplicateResourceException;
import com.hrry6.app.modules.catalog.dto.BookResponse;
import com.hrry6.app.modules.catalog.dto.CreateBookRequest;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;

@ExtendWith(MockitoExtension.class)
class BookServiceTest {

    @Mock
    private BookRepository bookRepository;

    @InjectMocks
    private BookService bookService;

    @Test
    void createBook_WhenIsbnIsUnique_ShouldSaveAndReturnBookResponse() {
        // Arrange
        CreateBookRequest request = new CreateBookRequest(
                "Clean Architecture",
                "Robert C. Martin",
                "9780134494166",
                5
        );

        Book savedEntity = Book.builder()
                .id(UUID.randomUUID())
                .title(request.title())
                .author(request.author())
                .isbn(request.isbn())
                .stockAvailable(5)
                .status(BookStatus.AVAILABLE)
                .build();

        given(bookRepository.existsByIsbn(request.isbn())).willReturn(false);
        given(bookRepository.save(any(Book.class))).willReturn(savedEntity);

        // Act
        BookResponse response = bookService.createBook(request);

        // Assert
        assertThat(response.id()).isEqualTo(savedEntity.getId());
        assertThat(response.title()).isEqualTo("Clean Architecture");
        assertThat(response.status()).isEqualTo(BookStatus.AVAILABLE);
        verify(bookRepository).save(any(Book.class));
    }

    @Test
    void createBook_WhenIsbnAlreadyExists_ShouldThrowDuplicateResourceException() {
        // Arrange
        CreateBookRequest request = new CreateBookRequest(
                "Clean Architecture",
                "Robert C. Martin",
                "9780134494166",
                5
        );
        given(bookRepository.existsByIsbn(request.isbn())).willReturn(true);

        // Act & Assert
        assertThatThrownBy(() -> bookService.createBook(request))
                .isInstanceOf(DuplicateResourceException.class)
                .hasMessageContaining("9780134494166");

        verify(bookRepository, never()).save(any(Book.class));
    }
}
```

---

## Shared Testcontainers Base Class (`AbstractIntegrationTest.java`)

Using a singleton container pattern across integration test classes avoids spinning up a new Docker container per test file, dramatically speeding up `./mvnw test`:

```java
package com.hrry6.app;

import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.test.context.ActiveProfiles;
import org.testcontainers.containers.PostgreSQLContainer;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
@ActiveProfiles("test") // Loads application-test.properties (runs database/init/01_schema.sql)
public abstract class AbstractIntegrationTest {

    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine");

    static {
        POSTGRES.start();
    }
}
```

### How `@ServiceConnection` + `application-test.properties` Work Together

1. `POSTGRES.start()` boots an ephemeral PostgreSQL 16 Docker container once for the entire test suite.
2. `@ServiceConnection` automatically injects the container's dynamic JDBC URL, username, and password into Spring's `DataSource` (no `@DynamicPropertySource` needed).
3. `@ActiveProfiles("test")` activates `application-test.properties`, executing `file:database/init/01_schema.sql` and running `ddl-auto=validate` against the real PostgreSQL schema.

---

## API Integration Test with `MockMvc` (`BookIntegrationTest.java`)

Tests the entire HTTP pipeline: `SecurityFilterChain`, `@PreAuthorize` RBAC, `@Valid` DTO validation, `BookService`, `BookRepository`, and real PostgreSQL constraints.

```java
package com.hrry6.app.modules.catalog;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.hrry6.app.AbstractIntegrationTest;
import com.hrry6.app.modules.catalog.dto.CreateBookRequest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

class BookIntegrationTest extends AbstractIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private BookRepository bookRepository;

    @BeforeEach
    void cleanDatabase() {
        bookRepository.deleteAll();
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void createBook_AsAdminWithValidPayload_ShouldReturn201() throws Exception {
        CreateBookRequest request = new CreateBookRequest(
                "Designing Data-Intensive Applications",
                "Martin Kleppmann",
                "9781449373320",
                10
        );

        mockMvc.perform(post("/api/v1/books")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("\$.success").value(true))
                .andExpect(jsonPath("\$.data.title").value("Designing Data-Intensive Applications"))
                .andExpect(jsonPath("\$.data.isbn").value("9781449373320"));
    }

    @Test
    @WithMockUser(roles = "MEMBER")
    void createBook_AsNonAdminMember_ShouldReturn403Forbidden() throws Exception {
        CreateBookRequest request = new CreateBookRequest(
                "Effective Java",
                "Joshua Bloch",
                "9780134685991",
                3
        );

        mockMvc.perform(post("/api/v1/books")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isForbidden())
                .andExpect(jsonPath("\$.success").value(false));
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void createBook_WithInvalidPayload_ShouldReturn400ValidationError() throws Exception {
        CreateBookRequest invalidRequest = new CreateBookRequest(
                "",          // Violates @NotBlank
                "Author",
                "123",       // Violates @Pattern ISBN
                -5           // Violates @Min(0)
        );

        mockMvc.perform(post("/api/v1/books")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(invalidRequest)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("\$.success").value(false))
                .andExpect(jsonPath("\$.errors.title").exists())
                .andExpect(jsonPath("\$.errors.isbn").exists())
                .andExpect(jsonPath("\$.errors.stockAvailable").exists());
    }

    @Test
    void getAllBooks_PublicEndpoint_ShouldReturn200WithoutAuth() throws Exception {
        bookRepository.save(Book.builder()
                .title("Spring in Action")
                .author("Craig Walls")
                .isbn("9781617297571")
                .stockAvailable(4)
                .status(BookStatus.AVAILABLE)
                .build());

        mockMvc.perform(get("/api/v1/books"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("\$.data.content.length()").value(1))
                .andExpect(jsonPath("\$.data.content[0].title").value("Spring in Action"));
    }
}
```

---

## Testing Annotations & Assertions Cheatsheet

| Annotation / Helper | Purpose |
| :--- | :--- |
| `@ExtendWith(MockitoExtension.class)` | Enables `@Mock` and `@InjectMocks` in JUnit 5 without loading Spring |
| `@SpringBootTest` | Boots the complete `ApplicationContext` for end-to-end integration tests |
| `@AutoConfigureMockMvc` | Injects `MockMvc` to simulate HTTP requests without starting a real network socket |
| `@ServiceConnection` | Auto-wires Testcontainers (`PostgreSQLContainer`) connection properties into Spring Boot 3.1+ |
| `@ActiveProfiles("test")` | Loads `src/test/resources/application-test.properties` |
| `@WithMockUser(username = "...", roles = "ADMIN")` | Simulates an authenticated user in `SecurityContextHolder` (bypasses JWT token generation for controller/RBAC tests) |
| `@Sql("/scripts/seed.sql")` | Executes custom SQL seed scripts before/after individual test methods |
