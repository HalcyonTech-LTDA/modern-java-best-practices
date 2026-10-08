---
name: modern-gradle-conventions
description: >-
  Use this skill when configuring, refactoring, or optimizing Gradle builds for modern Java projects (Modern Java Standard Baseline):
  exclusive Kotlin DSL (.gradle.kts), Convention Plugins via composite builds (build-logic), Version Catalogs (libs.versions.toml),
  dependency discipline (api vs implementation, BOMs, bundles), Java Toolchains, Configuration Cache, JVM Test Suites,
  parallel test execution, Jacoco, Pitest mutation testing, and Groovy-to-Kotlin DSL migration.
---

# Modern Gradle Build Conventions & Engineering Standards

This skill provides comprehensive architectural patterns, production standards, and runbooks for modern enterprise Gradle builds using **Kotlin DSL (`.gradle.kts`)** exclusively.

---

## Core Engineering Principles

1. **Exclusive Kotlin DSL (`.gradle.kts`)**:
   - Write all build scripts, convention plugins, and settings exclusively in Kotlin DSL.
   - Ban new Groovy DSL (`.gradle`) scripts. Kotlin DSL guarantees compile-time type safety, instantaneous IDE auto-completion, and refactoring reliability.

2. **Convention Plugins via Composite Build (`build-logic`)**:
   - Ban script injection via `subprojects { ... }` or `allprojects { ... }` in root build files.
   - Encapsulate reusable configuration in precompiled script plugins inside an included build (`build-logic`). Apply via type-safe `plugins { id(...) }`.

3. **Centralized Version Catalogs (`gradle/libs.versions.toml`)**:
   - Declare all library versions, artifacts, plugins, and bundle groups in the Version Catalog.
   - Enforce ABI boundaries: use `api` for exposed contracts and `implementation` for internal details (`java-library`).
   - Standardize multi-dependency stacks using BOM platforms (`platform(...)`) and bundles.

4. **Maximum Build Performance & Toolchains**:
   - Isolate JDK execution from the Gradle daemon using **Java Toolchains**.
   - Enable and mandate **Configuration Cache** (`org.gradle.configuration-cache=true`) and **Build Cache** (`org.gradle.caching=true`).
   - Adhere strictly to **Task Configuration Avoidance** APIs (`tasks.register`, `tasks.named`, `Provider<T>`, `Property<T>`).

5. **Modern JVM Test Suites & Quality Tooling**:
   - Model test segregation natively with **JVM Test Suites** (`testing { suites { ... } }`) for unit and integration testing.
   - Optimize test speed with parallel fork execution (`maxParallelForks`).
   - Enforce automated quality gates: strict compiler flags (`-Xlint:all`, `-parameters`), Jacoco coverage verification, and Pitest mutation testing.

---

## Detailed Topic Guides

1. **Convention Plugins & Multi-Project Architecture**:
   - Setting up `build-logic` as a composite build with `includeBuild("build-logic")`.
   - Creating precompiled script plugins (`java-common-conventions.gradle.kts`, `spring-boot-conventions.gradle.kts`).
   - [Read Guide](./references/convention_plugins_and_structure.md)

2. **Version Catalogs & Dependency Discipline**:
   - Structuring `gradle/libs.versions.toml` (`[versions]`, `[libraries]`, `[bundles]`, `[plugins]`).
   - Type-safe accessors in Kotlin DSL (`libs.spring.boot`, `libs.bundles.testing`).
   - Enforcing `api` vs `implementation`, `compileOnly`, `runtimeOnly`, and BOM platforms.
   - [Read Guide](./references/version_catalogs_and_dependencies.md)

3. **Performance, Configuration Cache & Java Toolchains**:
   - `gradle.properties` performance flags (Configuration Cache, Build Cache, Parallel, VFS watching).
   - Java Toolchains configuration for reproducible builds.
   - Eliminating anti-patterns that break the Configuration Cache.
   - [Read Guide](./references/performance_cache_and_toolchains.md)

4. **JVM Test Suites & Automated Quality Tooling**:
   - Configuring standard and integration test suites using `JvmTestSuite`.
   - Test logging, parallel execution, and environment configuration.
   - Jacoco coverage thresholds and Pitest mutation testing integration.
   - [Read Guide](./references/jvm_test_suites_and_quality.md)

5. **Groovy DSL to Kotlin DSL Migration**:
   - Comprehensive translation dictionary: closures to lambdas, property access, string interpolations.
   - Step-by-step migration runbook for legacy Gradle builds.
   - [Read Guide](./references/groovy_to_kotlin_dsl_migration.md)
