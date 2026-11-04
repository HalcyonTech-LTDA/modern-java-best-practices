<div align="center">

# Modern Java Best Practices Plugin
### Modern Java 21+ Language Idioms, Screaming Clean Architecture & DOP

[![Antigravity Plugin](https://img.shields.io/badge/Antigravity-Plugin%20v1.0.0-4F46E5.svg)](https://antigravity.google)
[![License: MIT](https://img.shields.io/badge/License-MIT-10B981.svg)](./LICENSE)
[![Organization](https://img.shields.io/badge/Organization-HalcyonTech--LTDA-black.svg)](https://github.com/HalcyonTech-LTDA)

<p align="center">
  <b>Modern Java best practices, idioms, Object-Oriented & Data-Oriented Programming, stable Virtual Thread concurrency, Screaming Clean Architecture, deterministic testing, and high-performance Gradle build conventions (Modern Java Standard Baseline).</b>
</p>

</div>

---

## Installation & Verification

### Antigravity CLI (`agy`)

Install directly into your Antigravity environment via the CLI:

```bash
agy plugin install https://github.com/HalcyonTech-LTDA/modern-java-best-practices
```

Verify that the plugin is active and adheres to the strict official schema:

```bash
agy plugin list
agy plugin validate --strict ./modern-java-best-practices
```

---

## Production-Strict Policy

This plugin adheres strictly to **100% finalized and standardized Java language features and APIs**. It does not generate, recommend, or require preview flags (`--enable-preview`) in production code.

---

---

## What's Included (Component Catalog)

### Contextual Rules (`rules/`)
Automatically loaded to govern and protect project contexts:
- **[`modern-java-engineering.md`](rules/modern-java-engineering.md)**: Automatically enforces modern Java 21+ baseline standards, Screaming Clean Architecture, Data-Oriented Programming (records, pattern matching), strict anti-Lombok policy, Loom concurrency, deterministic testing, and Gradle Kotlin DSL build conventions.

### Procedural Skills (`skills/`)
On-demand actionable workflows and runbooks:
- **[`/modern-gradle-conventions`](skills/modern-gradle-conventions/SKILL.md)**: Use this skill when configuring, refactoring, or optimizing Gradle builds for modern Java projects (Modern Java Standard Baseline): exclusive Kotlin DSL (.gradle.kts), Convention Plugins via composite builds (build-logic), Version Catalogs (libs.versions.toml), dependency discipline (api vs implementation, BOMs, bundles), Java Toolchains, Configuration Cache, JVM Test Suites, parallel test execution, Jacoco, Pitest mutation testing, and Groovy-to-Kotlin DSL migration.
- **[`/modern-java-clean-architecture`](skills/modern-java-clean-architecture/SKILL.md)**: Use this skill when architecting, implementing, or refactoring Java services using Screaming Architecture, Clean Architecture, OOP and Data-Oriented Programming (DOP): feature-first package structure, cross-feature communication, pure domain models, framework-free application layer, rich OOP entities, Value Objects with records, sealed domain event and error types (Result<T, E>), inbound/outbound ports (Ports & Adapters), and framework-agnostic interactors.
- **[`/modern-java-concurrency`](skills/modern-java-concurrency/SKILL.md)**: Use this skill when designing, implementing, or optimizing multi-threaded, concurrent, or asynchronous Java applications: Virtual Threads per-task model, task orchestration with CompletableFuture and Virtual Thread executors, workload separation (I/O vs CPU), and diagnosing/preventing carrier thread pinning.
- **[`/modern-java-idioms`](skills/modern-java-idioms/SKILL.md)**: Use this skill when writing, modernizing, or refactoring Java code to utilize modern Java language features (Modern Java Standard Baseline): Object-Oriented Programming (OOP) & SOLID principles, Data-Oriented Programming (DOP), Records and Record Patterns, exhaustive switch expressions, Sealed Hierarchies, Unnamed Variables and Patterns, Sequenced Collections, Flexible Constructor Bodies, Stream Gatherers, Dependency Hygiene (Anti-Lombok), and the Foreign Function & Memory (FFM) API.
- **[`/modern-java-testing`](skills/modern-java-testing/SKILL.md)**: Use this skill when designing, writing, or optimizing automated tests in Java (Modern Java Standard Baseline): Clean Architecture test pyramid, In-Memory Fakes over Mocks, integration testing with Testcontainers and @ServiceConnection, API testing with RestAssured, anti-flakiness with Awaitility and Clock injection, mutation testing with Pitest, test data generation with Instancio, JUnit Jupiter patterns (@Nested, @TempDir, @Tag), AssertJ, and architectural fitness rules with ArchUnit.

---

## Directory Hierarchy

```text
modern-java-best-practices/
├── plugin.json              # Official Antigravity manifest (schema v1)
├── LICENSE                  # MIT License (Copyright 2026 HalcyonTech LTDA)
├── .gitignore               # System ignores & installed_version.json exclusions
├── .gitattributes           # Deterministic line endings (* text=auto eol=lf)
├── README.md                # Golden Standard documentation & runbook
├── assets/
│   └── logo.svg             # Official Halcyon Tech corporate vector symbol
├── rules/                   # Contextual rules discovered automatically
│   └── modern-java-engineering.md
├── skills/                  # Actionable procedural skills (Progressive Disclosure)
│   └── modern-gradle-conventions/
│       └── SKILL.md
│   └── modern-java-clean-architecture/
│       └── SKILL.md
│   └── modern-java-concurrency/
│       └── SKILL.md
│   └── modern-java-idioms/
│       └── SKILL.md
│   └── modern-java-testing/
│       └── SKILL.md
```

---

## 💡 Prompt Recipes & Use Cases

```markdown
- "Refactor this Java service to modern Java 21+ idioms and Data-Oriented Programming"
- "Implement Screaming Clean Architecture with pure domain models and sealed hierarchies"
- "Configure deterministic unit and integration tests using Testcontainers"
```

---

## 📄 License & Corporate Governance

Distributed under the **MIT License**. Copyright (c) 2026 **HalcyonTech LTDA**.  
Engineered with precision and excellence by [Halcyon Tech](https://github.com/HalcyonTech-LTDA).
