# `pom.xml` & Core Dependencies

[🏠 Home](../README.md) > [🍃 Spring Boot](./README.md)

---

## Reference `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="[http://maven.apache.org/POM/4.0.0](http://maven.apache.org/POM/4.0.0)" xmlns:xsi="[http://www.w3.org/2001/XMLSchema-instance](http://www.w3.org/2001/XMLSchema-instance)"
    xsi:schemaLocation="[http://maven.apache.org/POM/4.0.0](http://maven.apache.org/POM/4.0.0) [https://maven.apache.org/xsd/maven-4.0.0.xsd](https://maven.apache.org/xsd/maven-4.0.0.xsd)">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.5</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.hrry6</groupId>
    <artifactId>library-app</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>library-app</name>
    <description>Library Management System API</description>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- Database Driver -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- JJWT (Java JWT) -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>0.12.6</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>0.12.6</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>0.12.6</version>
            <scope>runtime</scope>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Testing Starter (Standard) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- Testcontainers (PostgreSQL + JUnit 5 + Spring Boot 3.1+ ServiceConnection) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-testcontainers</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## Structural Sections Explained

- **`<parent>` (`spring-boot-starter-parent`)**: Acts as a curated Bill of Materials (BOM). It pins compatible versions for hundreds of transitive libraries (Jackson, Hibernate, Tomcat, JUnit, etc.), so official `spring-boot-starter-*` dependencies do not need explicit `<version>` tags.
- **`<properties>`**: Defines build-level variables such as `<java.version>21</java.version>`, which configures `maven-compiler-plugin` source and target bytecode levels.
- **`<build>` (`spring-boot-maven-plugin`)**: Repackages compiled classes and dependencies into a single runnable fat JAR (`java -jar target/library-app-0.0.1-SNAPSHOT.jar`). Excluding `lombok` strips compile-only annotation processing code from the final production JAR to keep image sizes smaller.

---

## Dependency Scopes

| Scope / Flag | Behavior |
| :--- | :--- |
| `compile` *(default)* | Available during compilation, testing, and packaged inside the final JAR |
| `runtime` | Not exposed to application source code at compile time, but bundled for runtime execution (prevents coupling code to driver internals) |
| `test` | Only available under `src/test/java`; completely excluded from production builds |
| `<optional>true</optional>` | Prevents downstream projects from transitively inheriting this dependency |

---

## Essential Dependencies Breakdown

### Core Application Starters

- **`spring-boot-starter-web`**: Provides Spring MVC (`@RestController`), Jackson for JSON serialization/deserialization, and an embedded Apache Tomcat HTTP server.
- **`spring-boot-starter-data-jpa`**: Integrates Hibernate ORM, Spring Data repositories (`JpaRepository`), transaction management (`@Transactional`), and HikariCP connection pooling.
- **`spring-boot-starter-security`**: Adds servlet filter-chain security, password hashing (`BCryptPasswordEncoder`), authentication managers, and method-level RBAC (`@PreAuthorize`).
- **`spring-boot-starter-validation`**: Enables Jakarta Bean Validation (`@Valid`, `@NotNull`, `@NotBlank`, `@Email`, `@Size`) powered by Hibernate Validator.

### Persistence & Auth Libraries

- **`org.postgresql:postgresql` (`runtime`)**: Official PostgreSQL JDBC driver used by HikariCP and Hibernate to communicate with PostgreSQL at runtime.
- **`io.jsonwebtoken:jjwt-api` (`compile`)**: Exposes public interfaces (`Jwts`, `Claims`, `SecretKey`) used directly in `JwtService` to build and parse tokens.
- **`io.jsonwebtoken:jjwt-impl` (`runtime`)**: Internal cryptographic implementation (HMAC-SHA, RSA) loaded dynamically via Java ServiceLoader at runtime.
- **`io.jsonwebtoken:jjwt-jackson` (`runtime`)**: Bridges JJWT with Spring's Jackson `ObjectMapper` to serialize/deserialize JWT claims as JSON.
- **`org.projectlombok:lombok` (`optional`)**: Generates boilerplate getters, setters, constructors, and builders (`@Getter`, `@Setter`, `@RequiredArgsConstructor`, `@Builder`) at compile time.

### Testing & Integration Suite

- **`spring-boot-starter-test` (`test`)**: All-in-one test bundle including JUnit 5, Mockito, AssertJ, Hamcrest, JsonPath, and `@SpringBootTest` / `@WebMvcTest` / `@DataJpaTest`.
- **`spring-security-test` (`test`)**: Adds helpers like `@WithMockUser` and `SecurityMockMvcRequestPostProcessors` to simulate authenticated users and roles in MockMvc tests.
- **`spring-boot-testcontainers` (`test`)**: Provides `@ServiceConnection`, automatically wiring dynamic Testcontainers ports and credentials into Spring's `DataSource` without manual `@DynamicPropertySource` boilerplate.
- **`org.testcontainers:junit-jupiter` & `postgresql` (`test`)**: Manages the lifecycle of disposable Dockerized PostgreSQL instances during integration tests (`@Testcontainers`, `@Container`).
