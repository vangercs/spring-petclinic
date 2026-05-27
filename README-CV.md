## Maven Wrapper and Build Standardization

This project uses the Maven Wrapper (`mvnw`) to ensure consistent Maven execution across local development environments and CI/CD runners.

One of the most common challenges in enterprise environments is the classic:

> "It works on my machine."

By versioning the Maven Wrapper directly inside the repository, the project controls the exact Maven version used during builds, regardless of the developer workstation, GitHub Actions runner, or container image executing the pipeline.

This becomes especially important in large-scale enterprise environments with hundreds of microservices, where build standardization and reproducibility are critical.

### Enterprise Build Standardization Strategy

- **Maven Wrapper** controls the Maven version
- **Parent POM / BOM** controls dependency and plugin versions
- **CI runner image** controls the Java/JDK version
- **Nexus / Artifactory** controls artifact repositories and dependency sourcing
- **Pipeline templates** standardize CI/CD execution across services

### Build Example

```bash
./mvnw -B clean verify
```

### Maven Wrapper Configuration

The project currently uses:

- Maven Wrapper version: `3.3.4`
- Apache Maven version: `3.9.12`

The Maven Wrapper automatically downloads the required Maven distribution if it is not already available on the system, ensuring consistent builds across developer workstations and CI/CD runners.

Using the Maven Wrapper improves:
- Build reproducibility
- CI/CD consistency
- Onboarding simplicity
- Cross-environment reliability
- Long-term maintainability of microservice platforms

## Dockerfile

This project uses a production-style **multi-stage Docker build** for the Spring Boot application.

The goal is not only to run the application in a container, but also to demonstrate common DevOps practices used in real CI/CD and Kubernetes environments.

```dockerfile
FROM eclipse-temurin:17-jdk AS build

WORKDIR /app

COPY .mvn .mvn
COPY mvnw .
COPY pom.xml .

RUN ./mvnw dependency:go-offline -B

COPY src src

RUN ./mvnw clean package -DskipTests

FROM eclipse-temurin:17-jre

WORKDIR /app

COPY --from=build /app/target/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

### Why Eclipse Temurin Java 17?

```dockerfile
FROM eclipse-temurin:17-jdk AS build
```

The build stage uses **Eclipse Temurin Java 17 JDK**.

Java 17 is a Long-Term Support release and is commonly used for modern Spring Boot applications. Eclipse Temurin is a trusted OpenJDK distribution frequently used in enterprise Docker and Kubernetes environments.

The `jdk` image is used in the build stage because the application needs Java build tools such as `javac` and Maven execution support.

---

### Multi-stage build

```dockerfile
FROM eclipse-temurin:17-jdk AS build
...
FROM eclipse-temurin:17-jre
```

This Dockerfile separates the **build environment** from the **runtime environment**.

The first stage compiles the application. The second stage contains only what is needed to run the application.

Benefits:

* smaller final image
* reduced attack surface
* faster image pulls
* cleaner runtime container
* no Maven, source code, or build cache in the final image

This is a common production pattern for Java applications running in Kubernetes.

---

### Maven Wrapper

```dockerfile
COPY .mvn .mvn
COPY mvnw .
COPY pom.xml .
```

The project uses the **Maven Wrapper** instead of relying on Maven being installed inside the image or on the build machine.

This improves reproducibility because the project controls the Maven version used during the build.

This is useful in CI/CD pipelines where builds should behave consistently across developer laptops, build agents, and Docker environments.

---

### Dependency caching optimization

```dockerfile
RUN ./mvnw dependency:go-offline -B
```

This command downloads Maven dependencies and plugins before the application source code is copied.

The goal is to take advantage of Docker layer caching.

If only application code changes, Docker can reuse the dependency layer instead of downloading all dependencies again.

This solves a very common production build problem:

* slow Maven dependency downloads
* unreliable external repository access
* repeated downloads in CI/CD pipelines
* unnecessary build time when only source code changed

The `-B` flag runs Maven in batch mode, which is preferred for automated builds.

---

### Copy source code after dependencies

```dockerfile
COPY src src
```

Source code changes more often than dependency definitions.

By copying `src` only after the Maven dependency step, the Docker build keeps dependency caching effective.

This is intentional and helps make local and CI/CD builds faster.

---

### Build the application

```dockerfile
RUN ./mvnw clean package -DskipTests
```

This command builds the Spring Boot application and creates the executable JAR under the `target` directory.

`-DskipTests` skips test execution during the Docker image build. In a real CI/CD pipeline, tests are usually executed in an earlier stage, while the Docker build stage focuses on packaging the already-validated application.

Typical pipeline flow:

```text
Unit tests → Security scans → Package → Docker build → Deploy
```

---

### Runtime image

```dockerfile
FROM eclipse-temurin:17-jre
```

The final image uses a **JRE** instead of a full JDK because the application only needs to run, not compile.

This keeps the runtime image smaller and more secure.

---

### Copy only the built artifact

```dockerfile
COPY --from=build /app/target/*.jar app.jar
```

Only the final JAR file is copied from the build stage into the runtime image.

The final container does not include:

* source code
* Maven wrapper files
* downloaded dependencies cache
* build tools
* temporary build artifacts

This is cleaner and safer for production deployments.

---

### Expose application port

```dockerfile
EXPOSE 8080
```

This documents that the application listens on port `8080`.

`EXPOSE` does not publish the port by itself. The port is published when running the container, for example:

```bash
docker run -p 8080:8080 myapp
```

---

### Start the application

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
```

The container starts the Spring Boot application using the executable JAR.

The JSON array format is preferred because it provides better process and signal handling than shell form, which is important for graceful shutdowns in Docker and Kubernetes.

---

### Production improvements to consider later

Future production hardening could include:

* running the container as a non-root user
* adding Spring Boot Actuator health endpoints
* configuring Kubernetes readiness and liveness probes
* adding JVM memory settings for containers
* enabling structured JSON logging
* adding Prometheus metrics
* adding OpenTelemetry tracing
* scanning the image for vulnerabilities
* considering a distroless runtime image

---

### Summary

This Dockerfile demonstrates several real-world DevOps practices:

* multi-stage Docker builds
* Maven Wrapper usage
* Docker layer caching
* smaller runtime images
* CI/CD-friendly build design
* Kubernetes-ready container packaging
* separation of build-time and runtime concerns

These patterns help make builds faster, images smaller, and deployments more production-ready.
