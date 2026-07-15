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

## Kubernetes Ingress (NGINX)

### What is Kubernetes Ingress?

A Kubernetes `Ingress` exposes HTTP/HTTPS applications running inside the cluster to users outside the cluster.

Without Ingress, a Kubernetes `ClusterIP` service is only reachable internally:

```text
Pod → Service → only inside Kubernetes
```

During early local testing we used:

```bash
kubectl port-forward svc/petclinic 8080:80
```

This creates a temporary tunnel from the developer machine into Kubernetes.

For a more production-like setup, we use an Ingress Controller.

---

### Why NGINX Ingress Controller?

Kubernetes provides the `Ingress` API object, but it does not implement traffic routing by itself.

An **Ingress Controller** watches Ingress resources and configures an actual reverse proxy/load balancer.

This project uses `ingress-nginx` because:

- widely used Kubernetes ingress controller
- open source
- lightweight for local development
- supports production features:
  - TLS termination
  - host-based routing
  - path-based routing
  - traffic rules
  - annotations/customization

---

### Traffic Flow

```text
Browser
   |
   |
petclinic.local
   |
   |
127.0.0.1:80
   |
   |
kind Node Port Mapping
   |
   |
NGINX Ingress Controller
   |
   |
Ingress Rule
(host: petclinic.local)
   |
   |
Kubernetes Service
(ClusterIP)
   |
   |
Spring Boot PetClinic Pod
(container port 8080)
```

---

### Local kind setup

Because kind runs Kubernetes nodes as Docker containers, ports must be exposed when the cluster is created.

`kind-config.yaml`

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: petclinic-lab

nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
```

Create the cluster:

```bash
kind create cluster --config kind-config.yaml
```

---

### Install ingress-nginx

Install the NGINX ingress controller:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Wait for the controller:

```bash
kubectl wait \
  --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=180s
```

Verify:

```bash
kubectl get pods -n ingress-nginx
```

---

### Application Ingress Resource

`k8s/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1

kind: Ingress

metadata:
  name: petclinic

spec:
  ingressClassName: nginx

  rules:
    - host: petclinic.local
      http:
        paths:
          - path: /
            pathType: Prefix

            backend:
              service:
                name: petclinic
                port:
                  number: 80
```

Deploy:

```bash
kubectl apply -f k8s/ingress.yaml
```

---

### Local DNS Mapping

Since `petclinic.local` is not a real DNS record, map it locally.

Edit:

```bash
sudo nano /etc/hosts
```

Add:

```text
127.0.0.1 petclinic.local
```

This simulates production DNS resolution.

Example production equivalent:

```text
petclinic.company.com
        |
        |
     Public DNS
        |
        |
 Cloud Load Balancer
        |
        |
 Ingress Controller
```

---

### Testing

Browser:

```text
https://petclinic.local
```

Command line:

```bash
curl https://petclinic.local
```

---

### Troubleshooting

Check Ingress:

```bash
kubectl get ingress
kubectl describe ingress petclinic
```

Check Service routing:

```bash
kubectl get svc
kubectl get endpoints petclinic
```

Check NGINX controller:

```bash
kubectl logs \
  -n ingress-nginx \
  deploy/ingress-nginx-controller
```

## Kubernetes ConfigMap

A ConfigMap is a Kubernetes object used to externalize application configuration from the container image.

Instead of rebuilding the Docker image for every configuration change, values can be injected into the container at runtime.

Typical use cases:

- Environment-specific configuration
- Feature flags
- Logging configuration
- Application properties
- Non-sensitive configuration values

> Sensitive values like passwords, tokens and certificates should be stored in Kubernetes Secrets, not ConfigMaps.

---

## ConfigMap as Environment Variables

A ConfigMap can expose values as container environment variables.

Example ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: petclinic-config
data:
  SPRING_PROFILES_ACTIVE: local
  LOG_LEVEL: INFO
```

Deployment:

```yaml
envFrom:
  - configMapRef:
      name: petclinic-config
```

Inside the container:

```bash
kubectl exec -it <pod-name> -- env
```

Example output:

```bash
SPRING_PROFILES_ACTIVE=local
LOG_LEVEL=INFO
```

Environment variables are useful for:

- Small key/value configuration
- Runtime overrides
- Environment selection

Examples:

```text
SPRING_PROFILES_ACTIVE=prod
JAVA_OPTS=-Xmx512m
ENVIRONMENT=aks-prod
```

---

# ConfigMap as Volume Mount

A ConfigMap can also be mounted into the container filesystem as files.

This is useful when applications expect configuration files.

Examples:

- application.properties
- application.yaml
- logback.xml
- nginx.conf
- prometheus.yaml

Example ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: petclinic-config-file

data:
  application.properties: |
    spring.profiles.active=test
    logging.level.root=INFO
```

Deployment:

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /config

volumes:
  - name: config-volume
    configMap:
      name: petclinic-config-file
```

Inside the container:

```bash
kubectl exec -it <pod-name> -- ls /config
```

Output:

```text
application.properties
```

Check the content:

```bash
kubectl exec -it <pod-name> -- cat /config/application.properties
```

---

# Spring Boot Configuration Loading

Spring Boot automatically checks external configuration locations:

```text
/config/application.properties
./config/application.properties
application.properties inside the jar
```

This allows Kubernetes to override packaged application configuration without rebuilding the Docker image.

Example:

Container image contains:

```properties
spring.profiles.active=dev
```

ConfigMap volume provides:

```properties
spring.profiles.active=test
```

Application starts with:

```text
Active profile: test
```

---

# Configuration Precedence

When the same property exists in multiple places, Spring Boot applies precedence rules.

Higher priority wins:

```text
Command-line arguments
        |
        v
Environment variables
        |
        v
External configuration files (/config)
        |
        v
Packaged application.properties
```

Example:

Environment variable ConfigMap:

```bash
SPRING_PROFILES_ACTIVE=local
```

Mounted ConfigMap file:

```properties
spring.profiles.active=test
```

Application result:

```text
The following profile is active: local
```

The environment variable wins.

---

# ConfigMap Updates

## Environment Variable ConfigMaps

Environment variables are injected when the container starts.

Updating the ConfigMap:

```bash
kubectl edit configmap petclinic-config
```

does NOT update existing container environment variables.

The pod must be restarted:

```bash
kubectl rollout restart deployment petclinic
```

---

## Mounted ConfigMap Volumes

Mounted ConfigMaps are updated automatically by Kubernetes.

Example:

```bash
kubectl edit configmap petclinic-config-file
```

Eventually:

```bash
cat /config/application.properties
```

shows the new value.

However:

> The application may not automatically reload the configuration.

Spring Boot normally reads configuration only during startup.

For automatic reload:

- Spring Cloud Kubernetes
- File watchers
- Application-specific reload mechanisms

---

# Production Pattern

A common production setup:

```text
Docker Image
     |
     | contains defaults
     v
application.properties
     |
     | overridden by
     v
ConfigMap Volume
     |
     | overridden by
     v
Environment Variables
     |
     | overridden by
     v
Secrets / runtime overrides
```

Example:

ConfigMap Volume:

```text
/config/application.yaml
/config/logback.xml
```

ConfigMap Environment Variables:

```text
SPRING_PROFILES_ACTIVE=prod
JAVA_OPTS=-Xmx1024m
```

Secrets:

```text
DATABASE_PASSWORD
API_TOKEN
CERTIFICATES
```

This keeps the Docker image immutable while allowing each environment to provide its own configuration.

## Kubernetes Secrets

A Secret is a Kubernetes object used to store sensitive configuration separately from the container image.

Secrets are commonly used for:

- Database usernames/passwords
- API keys
- Tokens
- Certificates
- Private keys

Secrets work very similarly to ConfigMaps:

- They can be injected as environment variables
- They can be mounted as files
- They allow configuration changes without rebuilding Docker images

> Important: Kubernetes Secrets are base64 encoded by default. Base64 is not encryption.

---

# Creating a Secret

Example Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: petclinic-secret

type: Opaque

data:
  DB_USERNAME: cGV0Y2xpbmlj
  DB_PASSWORD: cGFzc3dvcmQ=
```

The values are base64 encoded:

```text
cGV0Y2xpbmlj  -> petclinic
cGFzc3dvcmQ=  -> password
```

Create encoded values:

```bash
echo -n "petclinic" | base64
```

```bash
echo -n "password" | base64
```

The `-n` prevents adding a newline character before encoding.

Apply the Secret:

```bash
kubectl apply -f k8s/secret.yaml
```

Verify:

```bash
kubectl get secrets
```

Example:

```text
NAME                 TYPE      DATA
petclinic-secret     Opaque    2
```

---

# Inspecting Secrets

Describe the Secret:

```bash
kubectl describe secret petclinic-secret
```

Output:

```text
Name: petclinic-secret

Data
====
DB_USERNAME:  9 bytes
DB_PASSWORD:  8 bytes
```

Kubernetes hides Secret values from describe output.

Decode manually:

```bash
kubectl get secret petclinic-secret \
-o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

Output:

```text
password
```

---

# Inject Secret as Environment Variables

Secrets can be loaded into the container environment.

Deployment example:

```yaml
envFrom:
  - configMapRef:
      name: petclinic-config

  - secretRef:
      name: petclinic-secret
```

Restart the Deployment:

```bash
kubectl rollout restart deployment petclinic
```

Check inside the container:

```bash
kubectl exec -it <pod-name> -- env
```

Example:

```text
DB_USERNAME=petclinic
DB_PASSWORD=password
```

---

# Inject Individual Secret Values

A more controlled approach is selecting individual Secret keys.

Example:

```yaml
env:
  - name: DATABASE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: petclinic-secret
        key: DB_PASSWORD
```

Inside the container:

```bash
echo $DATABASE_PASSWORD
```

Output:

```text
password
```

This avoids exposing every Secret value to the container.

---

# Mount Secret as a Volume

Secrets can also be mounted as files.

Deployment:

```yaml
volumeMounts:
  - name: secret-volume
    mountPath: /secrets

volumes:
  - name: secret-volume
    secret:
      secretName: petclinic-secret
```

Inside the container:

```bash
kubectl exec -it <pod-name> -- ls /secrets
```

Example:

```text
DB_USERNAME
DB_PASSWORD
```

Each Secret key becomes a file.

Read the mounted Secret:

```bash
cat /secrets/DB_PASSWORD
```

Output:

```text
password
```

---

# Updating Secrets

Environment variable Secrets are loaded only when the container starts.

Updating a Secret:

```bash
kubectl edit secret petclinic-secret
```

does not update existing environment variables.

Restart required:

```bash
kubectl rollout restart deployment petclinic
```

---

Mounted Secrets behave differently.

Kubernetes updates the mounted files automatically:

```text
Secret changed
      |
      v
Mounted file updated
      |
      v
/secrets/DB_PASSWORD changes
```

However, applications usually need reload logic to use the new value.

---

# ConfigMap vs Secret

| Feature | ConfigMap | Secret |
|---|---|---|
| Purpose | Non-sensitive config | Sensitive config |
| Examples | profiles, logging, flags | passwords, tokens, certs |
| Storage format | plain text | base64 |
| Environment variables | yes | yes |
| Volume mount | yes | yes |
| Auto volume refresh | yes | yes |
| Encrypted by default | no | depends on cluster |

---

# Production Secret Management

In production, Secrets are often stored outside Kubernetes.

Example AKS pattern:

```text
Azure Key Vault
        |
        |
        v
Secrets Store CSI Driver
        |
        |
        v
Kubernetes Pod


Mounted Secret:
/mnt/secrets/database-password

Environment:
DATABASE_PASSWORD
```

Benefits:

- Kubernetes does not store the original secret
- Centralized secret rotation
- Access controlled with identities
- Audit logging

Common enterprise tools:

- Azure Key Vault
- HashiCorp Vault
- AWS Secrets Manager
- Google Secret Manager

---

# Typical Application Configuration Flow

```text
Docker Image
     |
     | contains defaults
     v

application.properties

     |
     | overridden by
     v

ConfigMap Volume

     |
     | overridden by
     v

ConfigMap Environment Variables

     |
     | combined with
     v

Secrets
```

This keeps application images immutable and allows each environment to provide its own secure configuration.

## PostgreSQL Persistent Storage

PostgreSQL stores its database files on a Kubernetes
PersistentVolumeClaim rather than in the container filesystem.

The PVC is mounted at:

`/var/lib/postgresql/data`

This allows database data to survive PostgreSQL pod recreation.

### Verify persistence

1. Create an owner in PetClinic.
2. Delete the PostgreSQL pod.
3. Wait for Kubernetes to create a replacement.
4. Confirm that the previously created owner still exists.

> Note: Local kind storage protects against pod recreation, but deleting
> the entire kind cluster may also delete the underlying development data.