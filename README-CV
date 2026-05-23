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