# 🏗️ Dockerfile Best Practices

[🏠 Home](../README.md) > [🐳 Docker](./README.md)

## 📖 Core Instructions Reference

| Instruction | Purpose | Key Note / Example |
| :--- | :--- | :--- |
| `FROM` | Sets the base image for the build stage | Must be the first instruction (`FROM eclipse-temurin:21-jre-alpine`) |
| `WORKDIR` | Sets the working directory for subsequent commands | Creates the folder automatically if it doesn't exist; use instead of `RUN cd ...` |
| `COPY` | Copies files/folders from host into the image | Preferred over `ADD` (use `ADD` only when auto-extracting local `.tar` archives) |
| `RUN` | Executes shell commands **during image build** | Each `RUN` creates a new cached layer (`RUN apt-get update && apt-get install -y curl`) |
| `ARG` | Defines build-time variables | Available only during `docker build` (`--build-arg`), not inside running containers |
| `ENV` | Sets persistent environment variables | Available both during build and inside the running container (`ENV NODE_ENV=production`) |
| `USER` | Sets the user/UID for subsequent instructions | Switch away from `root` before `ENTRYPOINT`/`CMD` for security |
| `EXPOSE` | Documents the port the container listens on | Informational only—you still need `-p 8080:8080` at runtime to publish it |
| `ENTRYPOINT` | Configures the container's main executable | Hard to override; use *exec form* `["java", "-jar", "app.jar"]` so signals (`SIGTERM`) work properly |
| `CMD` | Provides default arguments to `ENTRYPOINT` (or default command) | Easily overridden by passing arguments at the end of `docker run` |

## 🚀 Multi-Stage Build Pattern

Separates build dependencies (compilers, SDKs) from the final runtime image to drastically reduce image size and attack surface.

```dockerfile
# --- Stage 1: Build ---
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app

# Copy dependency files first to leverage Docker layer caching
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .
RUN ./mvnw dependency:go-offline

# Copy source code and compile
COPY src ./src
RUN ./mvnw clean package -DskipTests

# --- Stage 2: Runtime ---
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Run as a non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=builder /app/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 🛡️ Essential `.dockerignore`

Prevents bloated build contexts and accidental leaks of secrets or local artifacts into the image.

```text
.git
.gitignore
.env
*.md
node_modules
target
dist
build
.idea
.vscode
```

## 💡 Key Optimization Rules

| Rule | Why it matters |
| :--- | :--- |
| **Order layers from least to most frequently changed** | Copy `package.json` / `pom.xml` before `src/` so dependencies aren't re-downloaded on every code edit. |
| **Use `alpine` or `distroless` base images** | Cuts final image size from ~400MB+ down to ~50–150MB. |
| **Combine `RUN` commands with `&&`** | Reduces total image layers and ensures package manager caches are cleaned in the same layer. |
| **Never run as `root` in production** | Limits damage if an attacker compromises the application process. |
