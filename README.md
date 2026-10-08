# Modern Java Best Practices Plugin (Modern Java Standard Baseline)

An Antigravity plugin providing rules, design guidelines, and procedural skills for modern Java development (Modern Java Standard Baseline), Object-Oriented Programming (OOP) & SOLID principles, Data-Oriented Programming (DOP), production-stable Virtual Thread concurrency, Screaming Architecture, framework-free Clean Architecture, and professional deterministic testing.

---

## Installation

Install directly in Antigravity via the CLI:

```bash
agy plugin install https://github.com/HalcyonTech-LTDA/modern-java-best-practices
```

Verify that the plugin is installed and ready:

```bash
agy plugin list
```

---
## Production-Strict Policy

This plugin adheres strictly to **100% finalized and standardized Java language features and APIs**. It does not generate, recommend, or require preview flags (`--enable-preview`) in production code.

---

## What's Included

### 1. Contextual Rules (`rules/modern-java.md`)
Automatically enforced across Java and Gradle files via `globs`:
- **Core Principles**: Native-First Java standard library, production-strict policy, immutability by default, and clarity over brevity.
- **OOP & SOLID Principles**:
  - Single Responsibility (SRP), Open/Closed (OCP via interfaces and sealed sum types), Liskov Substitution (LSP), Interface Segregation (ISP via granular use case ports), and Dependency Inversion (DIP).
  - Decision Matrix: When to choose classic OOP, DOP, or the pragmatic hybrid approach.
- **Data-Oriented Programming (DOP)**: Modeling transparent data with `record`s, algebraic sum types with `sealed interface`, exhaustive `switch` pattern matching, compact constructors, and unnamed variables (`_`).
- **Code Style & Dependency Hygiene (Anti-Lombok)**:
  - Strict ban on Lombok boilerplate annotations (`@Data`, `@Value`, `@Getter`, `@Setter`, `@NoArgsConstructor`, `@AllArgsConstructor`, `@Builder`).
  - Explicit top-level `import` declarations over inline Fully/Package-Qualified Class Names (FQCN/PQCN).
  - Strict guidelines for `var` (type inference only when RHS is obvious).
  - Native `Stream.toList()` and Stream Gatherers (`Stream.gather`).
  - Multiline Text Blocks (`"""`) and interpolation with `String.formatted()`.
  - Standard library utilities (`java.util.Objects`, `java.nio.file.Files`, `java.time`) over Apache Commons or Guava.
- **Project Loom & Stable Concurrency**:
  - Unpooled virtual threads for I/O tasks (`Executors.newVirtualThreadPerTaskExecutor()`).
  - Production subtask orchestration via `CompletableFuture` combined with virtual thread executors.
  - Carrier thread pinning elimination: replacing blocking `synchronized` sections with `ReentrantLock`.
- **Screaming Clean Architecture & Cross-Feature Communication**:
  - Top-level packages scream business features (`ordering`, `billing`, `inventory`), organizing Clean Architecture layers internally.
  - Both `domain` AND `application` layers are 100% pure Java (zero framework annotations or dependencies). Frameworks and Spring bean declarations are quarantined exclusively in `infrastructure`.
  - Decoupled Cross-Feature Communication: Zero direct entity sharing; synchronous calls use Outbound Port + ACL Adapter invoking neighboring Inbound Ports; asynchronous side-effects use immutable Domain Events.
- **Advanced Performance & Encapsulation**: Foreign Function & Memory (FFM) API (`java.lang.foreign.*`) replacing JNI/Unsafe, direct `ByteBuffer` and channel I/O, and JPMS encapsulation with `MethodHandles`.
- **Modern Testing & Anti-Flakiness Standards**:
  - JUnit 5 exclusivity (prohibiting JUnit 4). BDD structure with `@Nested`, clean naming with `@DisplayNameGeneration(ReplaceUnderscores.class)`, and file isolation with `@TempDir`.
  - AssertJ fluent style with `assertThatThrownBy` and `SoftAssertions.assertSoftly`.
  - Layered testing: pure unit tests for domain (zero mocks), In-Memory Fakes for application use cases, and Testcontainers for infrastructure.
  - RestAssured for HTTP endpoints against dynamic ports with reusable specifications and record deserialization.
  - Singleton Container Pattern with Spring Boot `@ServiceConnection`; ban on `@DirtiesContext`.
  - Strict ban on `Thread.sleep()`; mandate Awaitility and `java.time.Clock` injection.
  - Mutation testing with Pitest, test data generation with Instancio, and architecture fitness testing with ArchUnit.
- **Modern Gradle Standards (Kotlin DSL & Build Logic)**:
  - Exclusive Kotlin DSL (`.gradle.kts`); prohibiting new Groovy DSL (`.gradle`) scripts.
  - Convention plugins via `build-logic` composite build; strict ban on `subprojects { ... }` and `allprojects { ... }`.
  - Version Catalogs (`gradle/libs.versions.toml`) for all dependencies, BOM platforms (`platform(...)`), and bundles.
  - ABI boundary enforcement (`api` vs `implementation`).
  - Java Toolchains, Configuration Cache (`org.gradle.configuration-cache=true`), Build Cache, and Task Configuration Avoidance.
  - JVM Test Suites (`testing { suites { ... } }`) for unit and integration tests, strict compiler flags, Jacoco, and Pitest.

### 2. On-Demand Skills (`skills/`)
- **`modern-java-idioms`**:
  - `oop_vs_dop_and_solid.md`: Comprehensive guide on OOP, SOLID principles, the DOP paradigm, the Expression Problem, and the OOP vs DOP vs Hybrid decision matrix.
  - `data_oriented_programming.md`: The 4 pillars of DOP in Java, algebraic data modeling, pattern matching without getters.
  - `dependency_hygiene_and_style.md`: Anti-Lombok migration, `var` usage rules, `Stream.toList()`, text blocks, and standard JDK utilities.
  - `pattern_matching_records.md`: Record patterns, `when` guards, and unnamed patterns.
  - `stream_gatherers.md`: Built-in gatherers (`windowFixed`, `windowSliding`, `scan`, `fold`) and custom gatherers.
  - `sequenced_collections_and_constructors.md`: Sequenced collections contracts and flexible constructor prologue validation before `super(...)`.
  - `ffm_and_buffers.md`: FFM API with `Arena`, `MemorySegment`, direct `ByteBuffer`, and JPMS modular encapsulation.
- **`modern-java-concurrency`**:
  - `virtual_threads.md`: Virtual thread execution model, Spring Boot configuration, and workload separation.
  - `completable_future_orchestration.md`: Production-stable subtask coordination, `allOf` aggregation, fail-fast mechanics, and timeout handling without preview APIs.
  - `pinning_and_locks.md`: Carrier thread pinning diagnosis via JVM flags and migration to `ReentrantLock`.
- **`modern-java-clean-architecture`**:
  - `screaming_architecture_and_cross_feature.md`: Screaming Clean Architecture package structure, cross-feature communication patterns (Outbound Port + ACL Adapter vs Domain Events), and modular boundary rules.
  - `domain_and_records.md`: Pure domain entities (OOP), value objects as records (DOP), and sealed `Result<T, E>` types.
  - `use_cases_and_ports.md`: Inbound and outbound ports, pure Java interactors, and constructor injection.
  - `framework_isolation_guide.md`: Strict framework isolation keeping `domain` and `application` pure Java, with Spring IoC configuration isolated in `infrastructure`.
- **`modern-java-testing`**:
  - `junit_jupiter_and_assertj_patterns.md`: Exclusivity of JUnit 5, `@Nested` BDD hierarchy, `@DisplayNameGeneration`, `@TempDir` filesystem isolation, and AssertJ soft assertions.
  - `rest_assured_api_testing.md`: RestAssured `given/when/then` DSL, reusable request/response specifications, and deserialization into Java records.
  - `testing_taxonomy_and_fakes.md`: Pure domain unit tests without mocks and In-Memory Fakes implementing Outbound Ports for use cases.
  - `testcontainers_and_integration.md`: Singleton Container Pattern, `@ServiceConnection`, database cleanup in `@BeforeEach`, and WireMock HTTP stubs.
  - `anti_flakiness_and_determinism.md`: Replacing `Thread.sleep()` with Awaitility, `java.time.Clock` injection, and resetting stateful resilience constructs.
  - `mutation_testing_and_archunit.md`: Mutation testing with Pitest, rich randomized fixtures with Instancio, and architecture fitness tests with ArchUnit.
- **`modern-gradle-conventions`**:
  - `convention_plugins_and_structure.md`: Setting up `build-logic` composite builds, precompiled script plugins, and eliminating root script injection.
  - `version_catalogs_and_dependencies.md`: Version Catalog structure (`libs.versions.toml`), `api` vs `implementation`, BOM platforms, and bundles.
  - `jvm_test_suites_and_quality.md`: JVM Test Suites for unit and integration tests, parallel fork execution, Jacoco verification thresholds, and Pitest.
  - `performance_cache_and_toolchains.md`: Java Toolchains, Configuration Cache, Build Cache, and Task Configuration Avoidance.
  - `groovy_to_kotlin_dsl_migration.md`: Groovy DSL to Kotlin DSL syntax comparison Rosetta Stone and step-by-step migration runbook.

---

## Directory Layout

```text
modern-java-best-practices/
├── plugin.json
├── README.md
├── rules/
│   └── modern-java.md
└── skills/
    ├── modern-java-idioms/
    │   ├── SKILL.md
    │   └── references/
    │       ├── data_oriented_programming.md
    │       ├── dependency_hygiene_and_style.md
    │       ├── ffm_and_buffers.md
    │       ├── oop_vs_dop_and_solid.md
    │       ├── pattern_matching_records.md
    │       ├── sequenced_collections_and_constructors.md
    │       └── stream_gatherers.md
    ├── modern-java-concurrency/
    │   ├── SKILL.md
    │   └── references/
    │       ├── completable_future_orchestration.md
    │       ├── pinning_and_locks.md
    │       └── virtual_threads.md
    ├── modern-java-clean-architecture/
    │   ├── SKILL.md
    │   └── references/
    │       ├── domain_and_records.md
    │       ├── framework_isolation_guide.md
    │       ├── screaming_architecture_and_cross_feature.md
    │       └── use_cases_and_ports.md
    ├── modern-java-testing/
    │   ├── SKILL.md
    │   └── references/
    │       ├── anti_flakiness_and_determinism.md
    │       ├── junit_jupiter_and_assertj_patterns.md
    │       ├── mutation_testing_and_archunit.md
    │       ├── rest_assured_api_testing.md
    │       ├── testcontainers_and_integration.md
    │       └── testing_taxonomy_and_fakes.md
    └── modern-gradle-conventions/
        ├── SKILL.md
        └── references/
            ├── convention_plugins_and_structure.md
            ├── groovy_to_kotlin_dsl_migration.md
            ├── jvm_test_suites_and_quality.md
            ├── performance_cache_and_toolchains.md
            └── version_catalogs_and_dependencies.md
```

---

## Managing the Plugin

Because this plugin is installed in `~/.gemini/config/plugins/modern-java-best-practices`, it is discovered globally across all workspaces.

- **Status**: Enabled by default.
- **Validation**:
  ```bash
  agy plugin validate ~/.gemini/config/plugins/modern-java-best-practices
  ```
- **Disable/Enable**:
  ```bash
  agy plugin disable modern-java-best-practices
  agy plugin enable modern-java-best-practices
  ```

---

## License

This project is licensed under the [MIT License](LICENSE) - see the LICENSE file for details.
Copyright (c) 2026 Halcyon Tech.
