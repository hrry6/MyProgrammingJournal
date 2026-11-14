# Project Init & Maven Wrapper (`mvnw`)

[🏠 Home](../README.md) > [🍃 Spring Boot](./README.md)

---

## Project Initialization (`spring init`)

```bash
spring init \
  --build=maven \
  --java-version=21 \
  --dependencie=web,data-jpa,postgresql,lombok,security,validation,testcontainers \
  --groupId=com.hrry6 \
  --artifactId=something \
  --packaging=jar \
  something
```

### Flag Breakdown

| FLag | Value | Description |
| :--- | :--- | :--- |
| `--build` | `maven` | Generates a maven project (`pom.xml`) along with `./mvnw` wrapper scripts |
| `--java-version` | `21` | Targets Java 21 LTS |
| `--dependencies` | `web,data-jpa,....` | Comma-separated Spring Initializr starter IDs for `pom.xml` |
| `--groupId` | `com.hrry6` | Base Java package namespace (`src/main/java/com/hrry6/something`) |
| `--artifactId` | `something` | Project identifier used as the base name of generated `.jar` file |
| `--packaging` | `jar` | Packages the app as a self containered executable JAR with embedded Tomcat server |
| `<directory>` | `something` | Output folder name where the project scaffold will be extracted |

### Common Initializr Dependency IDs

- `web` - Spring Web (REST Controllers, Jackson JSON, embedded Tomcat)
- `data-jpa` - Spring Data JPA and Hibernate ORM
- `postgresql` - PostgreSQL JDBC Driver
- `security` - Spring Security
- `validation` - Jakarta Bean Validation (`@Valid`, `@NotBland`, etc)
- `lombok` - Lombok annotation processor
- `testcontainers` - Testcontainers support for integration testing

---

## Maven Wrapper (`./mvnw`) Cheatsheet

Always prefer `./mvnw` over a global `mvn` installation so builds use the exact Maven version pinned in `.mvn/wrapper/maven-wrapper.properties`.

### Core Lifecycle Commands

| Command | Description |
| :--- | :--- |
| `./mvnw clean` | Deletes the `target/` directory to clear stale compiled bytecode and artifacts |
| `./mvnw compile` | Compiles `src/main/java` source files into `target/classes` |
| `./mvnw test` | Compiles and runs all unit and integration tests under `src/test/java` |
| `./mvnw package` | Runs tests and packages the application into an executable `.jar` inside `target/` |
| `./mvnw verify` | Runs package plus any post-package integration verification checks |

### Useful Flags & Execution Options

```bash
# Run the application locally in dev mode
./mvnw spring-boot:run

# Run with a specific Spring profile (e.g., application-dev.properties)
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev

# Override application properties directly from CLI
./mvnw spring-boot:run -Dspring-boot.run.arguments="--server.port=9090"

# Build fat JAR while skipping test execution (common in Docker builds)
./mvnw clean package -DskipTests

# Build fat JAR while skipping both test compilation and execution
./mvnw clean package -Dmaven.test.skip=true

# Run a single test class
./mvnw test -Dtest=BookServiceTest

# Run a specific test method inside a class
./mvnw test -Dtest=BookServiceTest#shouldCreateBook

# Inspect full dependency tree (useful for debugging version conflicts)
./mvnw dependency:tree

# Force Maven to re-check and re-download remote dependencies/snapshots
./mvnw clean install -U

# Run build strictly offline using local ~/.m2 cache
./mvnw test -o
```
