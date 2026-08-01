# Quarkus Project Definition

This document outlines the architectural patterns, libraries, and configurations for the Finance Control System MVP backend, built with Quarkus 3.33 and Java 25.

## 1. Core Technologies

*   **Java Version:** 25
*   **Quarkus Version:** 3.33.x
*   **Build/Dependency Tool:** Gradle (Kotlin DSL)
*   **Database:** PostgreSQL
*   **Authentication:** JWT (via SmallRye JWT)
*   **Primary Key Strategy:** UUID v7
*   **Auditing:** Hibernate Envers (via `quarkus-hibernate-envers`)
*   **Observability & Tracing:** OpenTelemetry (via `quarkus-opentelemetry`) integrated with Grafana (Loki & Tempo)


## 2. Dependency Management

We use Gradle with Kotlin DSL as our dependency manager. Quarkus projects manage dependencies using a Bill of Materials (BOM) to align compatible versions.

### 2.1. Gradle Properties (`gradle.properties`)

```properties
quarkusPlatformGroupId=io.quarkus.platform
quarkusPlatformArtifactId=quarkus-bom
quarkusPlatformVersion=3.33.0
```

### 2.2. Build Configuration (`build.gradle.kts`)

```kotlin
plugins {
    java
    id("io.quarkus")
}

repositories {
    mavenCentral()
}

val quarkusPlatformGroupId: String by project
val quarkusPlatformArtifactId: String by project
val quarkusPlatformVersion: String by project

dependencies {
    // Quarkus Platform BOM for version alignment
    implementation(enforcedPlatform("${quarkusPlatformGroupId}:${quarkusPlatformArtifactId}:${quarkusPlatformVersion}"))

    // REST and JSON (JAX-RS / RESTEasy Reactive)
    implementation("io.quarkus:quarkus-resteasy-reactive")
    implementation("io.quarkus:quarkus-resteasy-reactive-jackson")

    // Database and ORM
    implementation("io.quarkus:quarkus-hibernate-orm-panache")
    implementation("io.quarkus:quarkus-jdbc-postgresql")

    // Security and Authentication
    implementation("io.quarkus:quarkus-smallrye-jwt")
    implementation("io.quarkus:quarkus-security")

    // Validation
    implementation("io.quarkus:quarkus-hibernate-validator")

    // Auditing
    implementation("io.quarkus:quarkus-hibernate-envers")

    // Observability and Distributed Tracing (Grafana / OpenTelemetry)
    implementation("io.quarkus:quarkus-opentelemetry")


    // Testing
    testImplementation("io.quarkus:quarkus-junit5")
    testImplementation("io.rest-assured:rest-assured")
}

group = "br.com.bdws.financecontrol"
version = "1.0.0-SNAPSHOT"

java {
    sourceCompatibility = JavaVersion.VERSION_25
    targetCompatibility = JavaVersion.VERSION_25
}

tasks.withType<JavaCompile> {
    options.encoding = "UTF-8"
    options.compilerArgs.add("-parameters")
}
```

---

## 3. Project Structure and Architectural Patterns

The project follows a layered architecture, adapted for Quarkus standards.

```
src/main/java/com/financecontrol/system/
├── config/                 # CDI Producers and custom configuration providers
├── resource/               # JAX-RS REST Endpoints (replaces Controllers)
├── service/                # Business logic
├── repository/             # Data access layer (using Quarkus PanacheRepository)
├── entity/                 # Panache/JPA Entities (Database models)
├── model/                  # Domain models & aggregates (see domain-model-guidelines.md)
├── dto/                    # Data Transfer Objects (for API requests/responses)
├── vo/                     # Value Objects (for specific business values)
├── security/               # Custom security context or JWT filters
├── audit/                  # Custom audit listeners/revision entities
├── exception/              # Exception mappers (@ServerExceptionMapper)
└── util/                   # General utility classes
```

### 3.1. Entities (`br.com.bdws.financecontrol.entity`)

*   **Purpose:** Represent tables in the database. Annotated with `@Entity` from `jakarta.persistence`.
*   **Primary Key:** Use `java.util.UUID` as the type for primary keys, generated with UUID v7. Example:
    ```java
    @Id
    @Column(name = "id", updatable = false, nullable = false)
    private UUID id;
    ```
    *Note: Custom generator or database-side generation can be configured to produce UUID v7.*
*   **Auditing:** All entities requiring audit trails are annotated with `@Audited` from Hibernate Envers.
*   **Multi-Tenancy:** Entities include a `workspaceId` (UUID) for tenant isolation.
*   **Soft Delete:** Implement an `active` boolean flag for soft deletion instead of actual record removal.

### 3.2. Repositories (`br.com.bdws.financecontrol.repository`)

*   **Purpose:** Data Access Layer. Instead of Spring Data JPA, we extend `io.quarkus.hibernate.orm.panache.PanacheRepositoryBase<Entity, UUID>`.
*   **Custom Queries:** Leverage Panache's built-in query helpers, such as:
    ```java
    public List<MyEntity> findActiveByTenant(UUID workspaceId) {
        return list("workspaceId = ?1 and active = true", workspaceId);
    }
    ```

### 3.3. Services (`br.com.bdws.financecontrol.service`)

*   **Purpose:** Encapsulate business logic. Annotated with `@ApplicationScoped` (Jakarta CDI).
*   **Transaction Management:** Use `@Transactional` from `jakarta.transaction`.
*   **Validation:** Perform business rule validation.
*   **DTO Conversion:** Convert DTOs to entities and vice-versa.

### 3.4. Resources (`br.com.bdws.financecontrol.resource`)

*   **Purpose:** REST API endpoints using JAX-RS (instead of Spring `@RestController`). Annotated with `@Path`, `@Produces`, and `@Consumes`.
*   **HTTP Methods:** Use Jakarta REST annotations (`@GET`, `@POST`, `@PUT`, `@DELETE`).
*   **Input/Output:** Accept DTOs as input and return DTOs as output.
*   **Validation:** Use `@Valid` annotation on parameters for automatic validation via Hibernate Validator.
*   **Error Handling:** Use `@ServerExceptionMapper` inside a single `GlobalExceptionHandler` class. See [REST API Error Handling Guidelines](error-handling-guidelines.md) for strategy comparison, payload design, and blueprint code.
*   **Security:** Use `@RolesAllowed`, `@Authenticated`, or `@PermitAll` for endpoint-level authorization.

### 3.5. DTOs (`br.com.bdws.financecontrol.dto`)

*   **Purpose:** Data Transfer Objects. Used for request bodies and response payloads.
*   **Validation:** Apply `jakarta.validation` annotations (e.g., `@NotNull`, `@Size`, `@Pattern`).
*   **Immutability:** Prefer immutable DTOs, utilizing Java `record`.

### 3.6. Models (`br.com.bdws.financecontrol.model`)

*   **Purpose:** Represent pure domain-specific concepts, dynamic business calculations, and aggregate roots that do not directly map 1:1 to database tables.
*   **Architectural Boundaries:** Models are pure Java classes/records without framework annotations (`@Entity`, `@Transactional`) or repository dependencies. The `Service` layer orchestrates transactions and persists entities created by models.
*   **Architecture & Usage Guide:** For full sequence diagrams, layer responsibilities, multi-entity persistence patterns, and concrete code examples, see [Domain Model Architecture Guidelines](domain-model-guidelines.md).

### 3.7. Value Objects (`br.com.bdws.financecontrol.vo`)

*   **Purpose:** Small objects representing a descriptive aspect of the domain with no conceptual identity. Value Objects should be immutable.

---

## 4. Security Configuration

*   **Quarkus Security:** Stateless authentication via the SmallRye JWT extension.
*   **JWT Configuration:** Public keys, issuer URL, and authorization details are configured in `application.properties`:
    ```properties
    mp.jwt.verify.publickey.location=publickey.pem
    mp.jwt.verify.issuer=https://financecontrol.com/issuer
    ```
*   **Password Hashing:** Utilize Quarkus Security utilities (e.g., `BcryptUtil`) for password encoding and verification.
*   **CORS:** Configured directly via properties in `application.properties`:
    ```properties
    quarkus.http.cors=true
    quarkus.http.cors.origins=https://myfrontend.com
    ```

---

## 5. Other Common Configurations

*   **Error Handling:** Global exception handling using `@ServerExceptionMapper` methods inside `GlobalExceptionHandler` to return consistent `ErrorResponseDTO` payloads. Detailed comparison and implementation guidelines are in [error-handling-guidelines.md](error-handling-guidelines.md).
*   **Logging & Observability:** Integrated with **Grafana (Loki & Tempo)** using Quarkus OpenTelemetry (`quarkus-opentelemetry`). OpenTelemetry trace IDs are automatically injected into log MDC (`traceId`) and returned in `ErrorResponseDTO` (`Span.current().getSpanContext().getTraceId()`) for seamless trace-to-log correlation in Grafana dashboards.
*   **Multi-Tenancy:** Implement tenant filtering by reading the workspace header/JWT claim and applying Hibernate filters or Panache query decorators.

*   **UUID v7 Generation:** Set up a generator class or database triggers to populate the UUID fields with UUID v7 upon insertion.

---

## 6. Development Workflow (Gradle & Quarkus)

Quarkus provides a highly optimized developer experience.

### 6.1. Live Reload (Development Mode)
Run the following command to start Quarkus in development mode:
```bash
./gradlew quarkusDev
```
This enables live-reload, background testing, and the Quarkus Dev UI.

### 6.2. Running Tests
Run the project unit and integration tests:
```bash
./gradlew test
```

### 6.3. Packaging the Application
Build a runner jar or native executable:
*   **Standard Jar:**
    ```bash
    ./gradlew build
    ```
*   **Native Executable (GraalVM):**
    ```bash
    ./gradlew build -Dquarkus.package.type=native
    ```
