# Groovy DSL to Kotlin DSL Migration Guide

This guide provides a comprehensive syntax reference and step-by-step migration runbook for transitioning legacy Groovy build scripts (`.gradle`) to modern Kotlin DSL (`.gradle.kts`).

---

## 1. Syntax Comparison Rosetta Stone

| Feature | Groovy DSL (`build.gradle`) | Kotlin DSL (`build.gradle.kts`) |
| :--- | :--- | :--- |
| **Quotes** | Single `'text'` or double `"text"` | Double quotes only `"text"` |
| **Core Plugins** | `apply plugin: 'java'` | `plugins { java }` |
| **Community Plugins** | `id 'com.diffplug.spotless' version '6.25.0'` | `id("com.diffplug.spotless") version "6.25.0"` |
| **String Interpolation** | `"$group:$name:$version"` | `"$group:$name:$version"` or `"${libs.versions.java.get()}"` |
| **Dependency Declarations** | `implementation 'com.google.guava:guava:33.0.0-jre'` | `implementation("com.google.guava:guava:33.0.0-jre")` or `implementation(libs.guava)` |
| **Project Dependency** | `implementation project(':shared')` | `implementation(project(":shared"))` |
| **File References** | `file('src/main/resources')` | `file("src/main/resources")` |
| **Boolean Property Assignment** | `html.enabled = true` | `html.required.set(true)` |
| **Task Access** | `tasks.named('test') { ... }` | `tasks.named<Test>("test") { ... }` |
| **Task Registration** | `task customTask(type: Copy) { ... }` | `tasks.register<Copy>("customTask") { ... }` |
| **Extra Properties** | `ext.customProp = 'myValue'` | `extra["customProp"] = "myValue"` |
| **List Literal** | `['arg1', 'arg2']` | `listOf("arg1", "arg2")` |
| **Map Literal** | `[key: 'value']` | `mapOf("key" to "value")` |

---

## 2. Detailed Conversion Examples

### Plugins Block

**Groovy DSL (`build.gradle`):**
```groovy
plugins {
    id 'java'
    id 'jacoco'
    id 'org.springframework.boot' version '3.3.4'
}
```

**Kotlin DSL (`build.gradle.kts`):**
```kotlin
plugins {
    java
    jacoco
    id("org.springframework.boot") version "3.3.4"
    // Or via Version Catalog:
    // alias(libs.plugins.spring.boot)
}
```

---

### Task Configuration & Typed Containers

In Groovy, tasks are dynamically dispatched. In Kotlin DSL, use typed accessors or `named<T>()`:

**Groovy DSL (`build.gradle`):**
```groovy
tasks.withType(JavaCompile) {
    options.encoding = 'UTF-8'
    options.compilerArgs << '-parameters'
}

test {
    useJUnitPlatform()
    maxParallelForks = 4
}
```

**Kotlin DSL (`build.gradle.kts`):**
```kotlin
tasks.withType<JavaCompile>().configureEach {
    options.encoding = "UTF-8"
    options.compilerArgs.add("-parameters")
}

tasks.named<Test>("test") {
    useJUnitPlatform()
    maxParallelForks = 4
}
```

---

### Working with `Property<T>` and `Provider<T>`

In modern Gradle, many task properties use the lazy `Property` API. While Groovy permits direct assignment (`=`), Kotlin DSL requires `.set()`:

```kotlin
// Groovy DSL
reports.html.outputLocation = file("reports/tests")

// Kotlin DSL
reports.html.outputLocation.set(file("reports/tests"))
```

---

## 3. Step-by-Step Migration Runbook

Migrate incrementally to prevent breaking CI workflows:

1. **Step 1: Migrate `settings.gradle` $\rightarrow$ `settings.gradle.kts`**:
   - Rename `settings.gradle` to `settings.gradle.kts`.
   - Wrap strings in double quotes: `rootProject.name = "my-app"`.
   - Replace `include 'subproject'` with `include("subproject")`.
   - Run `./gradlew projects` to verify.

2. **Step 2: Introduce Version Catalog (`gradle/libs.versions.toml`)**:
   - Extract hardcoded dependency and version strings into `libs.versions.toml`.
   - This provides auto-complete for both Groovy and Kotlin scripts.

3. **Step 3: Create `build-logic` for Convention Plugins**:
   - Create `build-logic/` composite build using Kotlin DSL from day one.
   - Extract common compilation, testing, and toolchain configurations into precompiled script plugins.

4. **Step 4: Migrate Subproject Scripts Module by Module**:
   - Rename one subproject's `build.gradle` to `build.gradle.kts`.
   - Replace its contents with convention plugin applications and dependencies.
   - Run `./gradlew :subproject:check`.
   - Repeat across all modules.

5. **Step 5: Migrate Root `build.gradle` $\rightarrow$ `build.gradle.kts`**:
   - Once all subprojects use convention plugins, the root script can be stripped down to a minimal, clean Kotlin DSL file.
