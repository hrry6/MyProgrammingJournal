# Spring Boot Internals & General Annotations

[🏠 Home](../README.md) > [🍃 Spring Boot](./README.md)

---

## How Spring Boot Works Under the Hood

### The `@SpringBootApplication` Entry Point

Every Spring Boot app starts from a main class annotated with `@SpringBootApplication`, which is a shortcut combining three core annotations:

```java
@SpringBootApplication // = @SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan
public class LibraryAppApplication {
    public static void main(String[] args) {
        SpringApplication.run(LibraryAppApplication.class, args);
    }
}
```

1. **`@SpringBootConfiguration`**: Marks the class as the primary Spring configuration source (a specialized `@Configuration`).
2. **`@ComponentScan`**: Tells Spring to scan the current package (`com.hrry6.app`) and all its sub-packages for classes annotated with `@Component`, `@Service`, `@Repository`, `@RestController`, and `@Configuration`.
3. **`@EnableAutoConfiguration`**: Inspects the JARs on your classpath (`pom.xml`) and automatically configures default beans based on what libraries are present.

---

### Bootstrapping Flow (`SpringApplication.run`)

When you run `./mvnw spring-boot:run` or `java -jar app.jar`, Spring executes these steps in order:

```text
1. Load Environment       ──► Reads OS env vars, CLI args, and application.properties
       │
2. Create IoC Container   ──► Initializes ApplicationContext (the registry that holds all Beans)
       │
3. Component Scanning     ──► Finds your @Service, @Repository, @RestController, @Configuration classes
       │
4. Auto-Configuration     ──► Checks classpath & creates default beans (Tomcat, HikariCP, Jackson, etc.)
       │
5. Bean Wiring & Proxies  ──► Injects dependencies, wraps beans in AOP proxies, runs @PostConstruct
       │
6. Start Embedded Server  ──► Boots Tomcat on server.port (e.g., 8081) and listens for HTTP traffic
```

---

### How Auto-Configuration Works ("Opinionated Defaults")

Spring Boot uses `@ConditionalOn...` annotations internally so you don't have to write boilerplate XML or manual setup classes:

- Sees `spring-boot-starter-web` on classpath -> auto-configures **Embedded Tomcat** + **`DispatcherServlet`** + **Jackson `ObjectMapper`**.
- Sees `spring-boot-starter-data-jpa` + `postgresql` driver -> auto-configures **HikariCP `DataSource`** + **Hibernate `EntityManagerFactory`**.
- **Rule of Override:** Auto-configuration always backs off if you define your own custom `@Bean` (e.g., defining your own `SecurityFilterChain` replaces Spring Security's default login screen).

---

### How Spring Proxies Work (Crucial Mental Model)

When you put annotations like `@Transactional`, `@Async`, `@PreAuthorize`, or `@Cacheable` on a method, Spring does **not** modify your Java class directly. Instead, it wraps your bean inside a **Dynamic Proxy** at startup:

```text
Caller (e.g., Controller)
   │
   ▼
[ Spring Proxy ]  ──► Runs extra logic BEFORE/AFTER (e.g., Begin DB Transaction -> Commit/Rollback)
   │
   ▼
[ Real Service ]  ──► Executes your actual Java method
```

> **Important Gotcha (Self-Invocation):** If `methodA()` inside `BookService` calls `this.methodB()` (also inside `BookService`), the call goes directly to `this` and **bypasses the proxy**—meaning `@Transactional` or `@Async` on `methodB()` will be completely ignored.

---

## General & Cross-Layer Annotations

### A. Component Stereotypes (Registering Beans)

All stereotype annotations tell `@ComponentScan` to instantiate the class as a Spring-managed **Singleton Bean**. Technically they all behave like `@Component`, with extra semantic meaning:

| Annotation | Target Layer | Behavior & Extra Features |
| :--- | :--- | :--- |
| `@Component` | Any / Utility | Generic Spring-managed bean (e.g., `JwtService`, helpers, mappers, filters) |
| `@Service` | Business Logic | Marks domain/service layer; semantically indicates business rules |
| `@Repository` | Data Access | Marks persistence layer + automatically translates low-level JDBC/SQL exceptions into Spring's unified `DataAccessException` |
| `@RestController` | Web / API | Combines `@Controller` + `@ResponseBody` (automatically serializes return values to JSON) |
| `@Configuration` | Setup / Config | Marks a class that declares factory methods annotated with `@Bean` |

```java
@Configuration
public class AppConfig {
    // @Bean registers the return object of a method into the IoC Container
    // Used for third-party classes you cannot annotate with @Component
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

---

### B. Dependency Injection (DI) Annotations

Used to wire beans together across any layer:

| Annotation | Usage | Notes |
| :--- | :--- | :--- |
| `@Autowired` | Constructor / Field | Tells Spring to inject a matching bean. **Optional** if a class has only one constructor (standard when using Lombok `@RequiredArgsConstructor`) |
| `@Qualifier("beanName")` | Parameter / Field | Resolves ambiguity when multiple beans implement the same interface |
| `@Primary` | Bean Class / `@Bean` | Marks a bean as the default choice when multiple candidates exist |
| `@Lazy` | Bean / Injection Point | Delays bean creation until it is actually first used instead of at app startup |

```java
@Service
@RequiredArgsConstructor // Generates constructor for final fields -> Spring injects automatically
public class NotificationService {
    private final EmailSender emailSender; // Clean, immutable, easy to mock in tests
}
```

---

### C. Configuration & Environment Annotations

Used in any class (`@Component`, `@Service`, `@Configuration`) to read `application.properties` or control environment loading:

#### 1. `@Value` (Inject Single Property)

```java
@Component
public class JwtService {
    // Injects value from application.properties; supports ":default" fallback
    @Value("\${application.security.jwt.secret-key}")
    private String secretKey;

    @Value("\${application.security.jwt.expiration-ms:86400000}")
    private long jwtExpirationMs;
}
```

#### 2. `@ConfigurationProperties` (Type-Safe Group Binding)

Preferred over `@Value` when binding multiple related properties into a structured object:

```java
@Configuration
@ConfigurationProperties(prefix = "application.security.jwt")
@Getter
@Setter
public class JwtProperties {
    private String secretKey;
    private long expirationMs;
}
```

#### 3. `@Profile` (Environment-Specific Beans)

Restricts a bean or configuration class to load only when a specific profile is active (`spring.profiles.active`):

```java
@Component
@Profile("dev") // Loaded only in 'dev'; ignored in 'prod' or 'test'
public class DevDataSeeder implements CommandLineRunner {
    @Override
    public void run(String... args) { /* seed dummy books */ }
}
```

---

### D. Lifecycle & Cross-Cutting Execution Annotations

Annotations that control when code runs or modify how methods execute:

| Annotation | Package | Purpose |
| :--- | :--- | :--- |
| `@PostConstruct` | `jakarta.annotation` | Runs a method **once** immediately after the bean is created and dependencies are injected (e.g., warming up a cache or validating config) |
| `@PreDestroy` | `jakarta.annotation` | Runs cleanup logic right before the application shuts down and destroys the bean |
| `@Transactional` | `org.springframework.transaction.annotation` | Wraps method execution in a database transaction; rolls back automatically on unchecked exceptions (`RuntimeException`) |
| `@Async` | `org.springframework.scheduling.annotation` | Executes the method in a separate background thread pool (requires `@EnableAsync` on a `@Configuration` class) |
| `@Scheduled` | `org.springframework.scheduling.annotation` | Runs a method periodically via cron or fixed delay (requires `@EnableScheduling` on a `@Configuration` class) |

```java
@Component
@Slf4j
public class CacheWarmup {

    @PostConstruct
    public void init() {
        log.info("Bean initialized and dependencies injected!");
    }

    @Scheduled(cron = "0 0 2 * * *") // Runs daily at 02:00 AM
    public void cleanupExpiredTokens() {
        // Periodic background maintenance
    }
}
```
