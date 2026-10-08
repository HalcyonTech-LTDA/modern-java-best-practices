# Convention Plugins & Multi-Project Architecture

Modern Gradle architecture replaces brittle cross-project script injection with **Convention Plugins** implemented via **Composite Builds (`build-logic`)**.

---

## 1. The Anti-Pattern: `subprojects { ... }` and `allprojects { ... }`

In legacy Gradle builds, the root `build.gradle` routinely injected dependencies, plugins, and task configurations into child projects:

```kotlin
// 🚫 BANNED ANTI-PATTERN in Modern Gradle
allprojects {
    repositories { mavenCentral() }
}
subprojects {
    apply(plugin = "java")
    dependencies {
        "testImplementation"("org.junit.jupiter:junit-jupiter:5.10.0")
    }
}
```

### Why Script Injection is Banned
1. **Breaks Project Decoupling**: Subprojects cannot be understood or built in isolation; their behavior depends on implicit state injected from ancestor scripts.
2. **Violates Configuration Cache & Project Isolation**: Cross-project model mutation prevents Gradle from isolating projects during configuration, severely degrading parallel configuration performance.
3. **Execution Ordering Hazards**: Configuring child projects from the root introduces subtle ordering bugs when child projects configure further properties.

---

## 2. Modern Standard: Precompiled Script Plugins in `build-logic`

Instead of cross-script mutation, modern Gradle encapsulates reusable build logic into internal plugins located in an included build named `build-logic`.

### Directory Layout

```text
my-application/
├── build-logic/                           # Composite build for build conventions
│   ├── settings.gradle.kts                # Empty or configuring plugin repositories
│   ├── build.gradle.kts                   # Declares 'kotlin-dsl' plugin
│   └── src/main/kotlin/
│       ├── myproject.base-conventions.gradle.kts
│       ├── myproject.java-common-conventions.gradle.kts
│       └── myproject.application-conventions.gradle.kts
├── gradle/
│   ├── wrapper/
│   └── libs.versions.toml                 # Version Catalog
├── services/
│   ├── order-service/
│   │   └── build.gradle.kts               # Applies myproject.application-conventions
│   └── payment-service/
│       └── build.gradle.kts
├── settings.gradle.kts                    # Root settings: includeBuild("build-logic")
└── build.gradle.kts                       # Clean, minimal root script
```

---

## 3. Implementation Step-by-Step

### Step 1: Include `build-logic` in Root `settings.gradle.kts`

```kotlin
// settings.gradle.kts (Root)
rootProject.name = "my-application"

pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}

// Wire the composite build
includeBuild("build-logic")

include("services:order-service")
include("services:payment-service")
include("shared:domain-models")
```

### Step 2: Configure `build-logic/settings.gradle.kts` & `build-logic/build.gradle.kts`

```kotlin
// build-logic/settings.gradle.kts
rootProject.name = "build-logic"

// Share the main version catalog with build-logic
dependencyResolutionManagement {
    repositories {
        mavenCentral()
    }
    versionCatalogs {
        create("libs") {
            from(files("../gradle/libs.versions.toml"))
        }
    }
}
```

```kotlin
// build-logic/build.gradle.kts
plugins {
    `kotlin-dsl`
}

dependencies {
    // Add third-party plugins that convention plugins apply
    implementation(libs.plugins.pitest.asProvider())
    implementation(libs.plugins.spotless.asProvider())
}
```

### Step 3: Author Precompiled Script Plugins

Convention plugins are written directly in Kotlin DSL files matching `<plugin-id>.gradle.kts`:

```kotlin
// build-logic/src/main/kotlin/myproject.java-common-conventions.gradle.kts
plugins {
    java
    jacoco
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(21))
    }
}

tasks.withType<JavaCompile>().configureEach {
    options.encoding = "UTF-8"
    options.compilerArgs.addAll(listOf("-parameters", "-Xlint:all", "-Werror"))
}

testing {
    suites {
        val test by getting(JvmTestSuite::class) {
            useJUnitJupiter()
        }
    }
}
```

```kotlin
// build-logic/src/main/kotlin/myproject.library-conventions.gradle.kts
plugins {
    id("myproject.java-common-conventions")
    `java-library`
}
```

---

## 4. Applying Conventions in Subprojects

Subproject `build.gradle.kts` files become declarative, concise, and focused strictly on project-specific concerns:

```kotlin
// services/order-service/build.gradle.kts
plugins {
    id("myproject.library-conventions")
}

dependencies {
    api(project(":shared:domain-models"))
    implementation(libs.jackson.databind)
    testImplementation(libs.assertj.core)
}
```

---

## 5. Architectural Rules for Convention Plugins

1. **Namespace Plugin IDs**: Always prefix convention plugins with your organization or project name (e.g., `myproject.java-conventions`), never generic names like `java-conventions`.
2. **Layering & Composition**: Build fine-grained conventions that compose cleanly:
   - `myproject.base-conventions` (repositories, encoding)
   - `myproject.java-common-conventions` (toolchains, compiler flags, testing suites)
   - `myproject.library-conventions` (`java-library`, ABI management)
   - `myproject.application-conventions` (`application` or Spring Boot runner)
3. **No Direct Inter-Project Script Dependencies**: Never reference another project's tasks or internal configurations directly. Depend strictly on projects via the `dependencies { implementation(project(":name")) }` dependency container.
