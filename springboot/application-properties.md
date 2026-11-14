# Application Configuration (`application.properties`)

[🏠 Home](../README.md) > [🍃 Spring Boot](./README.md)

---

## Main Runtime (`src/main/resources/application.properties`)

```properties
# ==========================================
# APPLICATION & SERVER
# ==========================================
spring.application.name=library-app
server.port=8081

# ==========================================
# POSTGRESQL DATASOURCE
# ==========================================
spring.datasource.url=jdbc:postgresql://localhost:5432/lib-app
spring.datasource.username=lib-app
spring.datasource.password=lib-app123

# ==========================================
# JWT SECURITY CONFIGURATION
# ==========================================
application.security.jwt.secret-key=${JWT_SECRET:404E635266556A586E3272357538782F413F4428472B4B6250645367566B5970}
application.security.jwt.expiration-ms=86400000
```

### Property Breakdown

| Property | Value / Example | Purpose |
| :--- | :--- | :--- |
| `spring.application.name` | `library-app` | Logical name used in startup logs, Actuator metrics, and distributed tracing |
| `server.port` | `8081` | Overrides default Tomcat port (`8080`) to avoid collisions with frontend/other services |
| `spring.datasource.url` | `jdbc:postgresql://localhost:5432/lib-app` | JDBC connection string (`jdbc:postgresql://<host>:<port>/<database>`) |
| `spring.datasource.username` | `lib-app` | PostgreSQL database role/user |
| `spring.datasource.password` | `lib-app123` | PostgreSQL role password |
| `application.security.jwt.secret-key` | `${JWT_SECRET:404E...}` | Custom property injected via `@Value` or `@ConfigurationProperties`. Must be >= 256 bits (32+ bytes / 64 hex chars) for `HS256` |
| `application.security.jwt.expiration-ms` | `86400000` | Token validity duration in milliseconds (`86400000` ms = 24 hours) |

### Environment Variable Placeholder Syntax (`${VAR:default}`)

Spring resolves `${JWT_SECRET:fallback}` in order:

1. Checks OS environment variables (`export JWT_SECRET=...`) or Docker Compose `environment:`.
2. Checks JVM system properties (`-DJWT_SECRET=...`).
3. Falls back to the literal value after `:` if neither is defined—allowing zero-config local startup while keeping production secrets externalized.

### Recommended Production Additions

```properties
# Disable Open Session in View to prevent holding DB connections during JSON serialization
spring.jpa.open-in-view=false

# Use 'validate' in prod (schema managed by SQL scripts/Flyway); avoid 'update' in prod
spring.jpa.hibernate.ddl-auto=validate

# Never expose internal exception messages or stack traces in HTTP error responses
server.error.include-message=never
server.error.include-stacktrace=never
```

---

## Test Profile (`src/test/resources/application-test.properties`)

Activated by annotating integration test classes with `@ActiveProfiles("test")`. Spring merges `application.properties` with `application-test.properties`, overriding matching keys.

```properties
# ==========================================
# SQL SCHEMA INITIALIZATION
# ==========================================
spring.sql.init.mode=always
spring.sql.init.schema-locations=file:database/init/01_schema.sql

# ==========================================
# JPA / HIBERNATE VALIDATION
# ==========================================
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
```

### Property Breakdown

| Property | Value | Purpose |
| :--- | :--- | :--- |
| `spring.sql.init.mode` | `always` | Forces Spring Boot to execute SQL initialization scripts on every test context startup (default is `embedded`, which skips external PostgreSQL containers) |
| `spring.sql.init.schema-locations` | `file:database/init/01_schema.sql` | Path to the DDL script. `file:` resolves relative to the project root directory (outside `src/`), whereas `classpath:` resolves inside `resources/` |
| `spring.jpa.hibernate.ddl-auto` | `validate` | Instructs Hibernate to verify that all `@Entity` classes, column names, data types, and constraints strictly match `01_schema.sql` without modifying tables |
| `spring.jpa.show-sql` | `false` | Keeps test runner output clean by suppressing raw SQL statements in stdout |

### How This Pairs with Testcontainers

1. **Container Boot**: `@ServiceConnection` spins up a fresh, empty PostgreSQL Docker container and overrides `spring.datasource.*` dynamically.
2. **Schema Provisioning**: `spring.sql.init.mode=always` executes `database/init/01_schema.sql` against that blank container, guaranteeing tests run against the exact same DDL used in production.
3. **Entity Verification**: `ddl-auto=validate` immediately fails the test suite if any Java `@Entity` drifts out of sync with the SQL schema.
