# Version Catalogs & Dependency Discipline

Modern Gradle enforces single-source-of-truth dependency management using **Version Catalogs (`gradle/libs.versions.toml`)** and strict ABI boundary control (`api` vs `implementation`).

---

## 1. Version Catalog Structure (`gradle/libs.versions.toml`)

The Version Catalog centralizes all third-party coordinates, versions, plugins, and bundle declarations:

```toml
[versions]
java = "21"
spring-boot = "3.3.4"
jackson = "2.17.2"
junit-jupiter = "5.11.0"
assertj = "3.26.3"
testcontainers = "1.20.1"
instancio = "5.0.2"
pitest = "1.15.0"
pitest-plugin = "1.15.0"

[libraries]
# Platforms / BOMs
spring-boot-bom = { module = "org.springframework.boot:spring-boot-dependencies", version.ref = "spring-boot" }
jackson-bom = { module = "com.fasterxml.jackson:jackson-bom", version.ref = "jackson" }
testcontainers-bom = { module = "org.testcontainers:testcontainers-bom", version.ref = "testcontainers" }

# Domain & Application
jackson-databind = { module = "com.fasterxml.jackson.core:jackson-databind" }

# Testing
junit-jupiter = { module = "org.junit.jupiter:junit-jupiter", version.ref = "junit-jupiter" }
assertj-core = { module = "org.assertj:assertj-core", version.ref = "assertj" }
instancio-junit = { module = "org.instancio:instancio-junit", version.ref = "instancio" }
awaitility = { module = "org.awaitility:awaitility", version = "4.2.2" }
testcontainers-postgresql = { module = "org.testcontainers:postgresql" }

[bundles]
testing = ["junit-jupiter", "assertj-core", "instancio-junit", "awaitility"]

[plugins]
spring-boot = { id = "org.springframework.boot", version.ref = "spring-boot" }
pitest = { id = "info.solidsoft.pitest", version.ref = "pitest-plugin" }
```

### Naming Conventions & Type-Safe Accessors
Gradle converts kebab-case or underscore separators in TOML aliases into type-safe dot accessors in Kotlin DSL:
- `spring-boot-bom` $\rightarrow$ `libs.spring.boot.bom`
- `jackson-databind` $\rightarrow$ `libs.jackson.databind`
- `[bundles] testing` $\rightarrow$ `libs.bundles.testing`
- `[plugins] spring-boot` $\rightarrow$ `libs.plugins.spring.boot`

---

## 2. Dependency Configuration Semantics

In Java projects, applying the `java-library` plugin unlocks granular control over compilation boundaries:

| Configuration | When to Use | Transitive Exposure |
| :--- | :--- | :--- |
| **`api`** | Types exposed in public method signatures, return types, or class inheritance of your module. | Transitive: consumers of this module will also compile against these dependencies. |
| **`implementation`** | Internal implementation details (classes not exposed in public API contracts). | Private: consumers of this module do not see these dependencies at compile time. |
| **`compileOnly`** | Annotations, code generators, or tools required only at compile time (e.g. `@Nullable`). | Not included in runtime classpath. |
| **`runtimeOnly`** | JDBC drivers, logging implementations (Logback), or runtime plugins not needed during compilation. | Included only on the runtime classpath. |
| **`testImplementation`** | Testing frameworks, assertion libraries, and test fixtures (JUnit, AssertJ). | Scoped exclusively to the test source set. |
| **`testRuntimeOnly`** | Test runtime engines (e.g. `junit-platform-launcher`). | Scoped to test execution runtime. |

### The `api` vs `implementation` Golden Rule

```kotlin
// In shared:domain-models/build.gradle.kts
plugins {
    `java-library`
}

dependencies {
    // 🚫 BAD: Leaking internal parsing utility to downstream consumers
    // api(libs.jackson.databind)

    // ✅ GOOD: Implementation detail kept private; prevents recompile cascades
    implementation(libs.jackson.databind)
}
```

> [!IMPORTANT]
> Using `implementation` instead of `api` prevents recompilation cascades across multi-project builds. When an internal implementation dependency changes, downstream modules do not need to be recompiled.

---

## 3. Managing Platforms & BOMs

Use Gradle's native `platform(...)` mechanism to import Bill of Materials (BOMs). This eliminates the need to declare explicit versions for sub-artifacts:

```kotlin
dependencies {
    // Import BOM platform
    implementation(platform(libs.spring.boot.bom))
    implementation(platform(libs.jackson.bom))
    testImplementation(platform(libs.testcontainers.bom))

    // Versions are governed by the BOM
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("com.fasterxml.jackson.core:jackson-databind")
    testImplementation("org.testcontainers:postgresql")
}
```

- **`platform(...)`**: Participates in standard dependency resolution; if a direct dependency requests a higher version, the higher version wins.
- **`enforcedPlatform(...)`**: Acts like a Maven `<dependencyManagement>` override, forcing versions even if transitives request different versions (use sparingly).

---

## 4. Bundles for Common Stacks

Group tightly coupled dependencies into bundles to keep module `build.gradle.kts` files concise:

```kotlin
dependencies {
    // Single line pulls in JUnit Jupiter, AssertJ, Instancio, and Awaitility
    testImplementation(libs.bundles.testing)
}
```

---

## 5. Dependency Constraints & Conflict Resolution

Enforce global constraints or align transitives across all projects using constraints blocks in convention plugins:

```kotlin
dependencies {
    constraints {
        implementation(libs.commons.compress) {
            because("CVE-2024-XXXX remediation in transitive dependencies")
        }
    }
}
```

To fail the build immediately on any version conflict instead of Gradle's default "highest version wins":

```kotlin
configurations.all {
    resolutionStrategy {
        failOnVersionConflict()
    }
}
```
