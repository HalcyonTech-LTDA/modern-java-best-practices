---
name: modern-java-testing
description: >-
  Use this skill when designing, writing, or optimizing automated tests in Java (Modern Java Standard Baseline):
  Clean Architecture test pyramid, In-Memory Fakes over Mocks, integration testing with Testcontainers and @ServiceConnection,
  API testing with RestAssured, anti-flakiness with Awaitility and Clock injection, mutation testing with Pitest,
  test data generation with Instancio, JUnit Jupiter patterns (@Nested, @TempDir, @Tag), AssertJ, and architectural fitness rules with ArchUnit.
---

# Professional Automated Testing in Modern Java

This skill provides standards, tools, and runbooks for building fast, deterministic, and highly effective automated test suites in modern Java (Modern Java Standard Baseline).

## Core Principles

1. **Test Real Behavior**: Prioritize pure unit tests for domain invariants and In-Memory Fakes for application use cases over brittle Mockito verifications.
2. **Modern JUnit 5 & AssertJ Conventions**: Structure tests with `@Nested`, eliminate JUnit 4 imports, generate readable names with `@DisplayNameGeneration`, isolate files with `@TempDir`, and use AssertJ soft assertions.
3. **Deterministic & Flakeless**: Never use `Thread.sleep()`. Inject `java.time.Clock` for temporal control and use `Awaitility` for asynchronous assertions.
4. **Optimized Integration & API Testing**: Use the Singleton Container Pattern with Testcontainers and `@ServiceConnection`, avoid `@DirtiesContext`, and test REST endpoints using RestAssured with reusable specifications.
5. **Meaningful Coverage & Governance**: Target high mutation scores with Pitest and automate architectural governance with ArchUnit.

## Architectural Guides & Runbooks

- **JUnit Jupiter & AssertJ Conventions**: [JUnit Jupiter & AssertJ Patterns](./references/junit_jupiter_and_assertj_patterns.md)
  - Eliminating JUnit 4 legacy imports.
  - BDD context nesting with `@Nested`.
  - Sentence-style naming with `@DisplayNameGeneration(ReplaceUnderscores.class)`.
  - Zero-leak file testing with `@TempDir`.
  - Selective test filtering with `@Tag`.
  - Multi-field assertion reporting with AssertJ `assertSoftly`.
- **RestAssured API Testing**: [RestAssured API Testing Guide](./references/rest_assured_api_testing.md)
  - Fluent `given() -> when() -> then()` DSL.
  - Reusable `RequestSpecification` and `ResponseSpecification`.
  - Extracting and deserializing JSON payloads directly into Java `record`s.
  - Failure logging and random web port binding.
- **Taxonomy & In-Memory Fakes**: [Testing Taxonomy & Fakes](./references/testing_taxonomy_and_fakes.md)
  - Zero-mock unit tests for domain entities.
  - Designing In-Memory Fakes implementing Outbound Ports (SPIs) for use cases.
- **Integration Testing with Testcontainers**: [Testcontainers & Integration Guide](./references/testcontainers_and_integration.md)
  - Singleton container pattern across test suites.
  - Spring Boot `@ServiceConnection` dynamic binding.
  - In-place database cleanup in `@BeforeEach` without context reloads.
  - External API simulation with WireMock.
- **Eliminating Flaky Tests & Race Conditions**: [Anti-Flakiness & Determinism](./references/anti_flakiness_and_determinism.md)
  - Replacing `Thread.sleep()` with Awaitility polling.
  - Injecting `java.time.Clock` with `Clock.fixed()`.
  - Resetting circuit breakers, rate limiters, and distributed locks.
  - Dynamic random ports (`WebEnvironment.RANDOM_PORT`).
- **Meaningful Coverage & Governance**: [Mutation Testing, Instancio & ArchUnit](./references/mutation_testing_and_archunit.md)
  - Mutation testing with Pitest to kill test defects.
  - Realistic object graph generation with Instancio.
  - Parameterized boundary testing with `@ParameterizedTest`.
  - Automated architectural fitness testing with ArchUnit.

## Testing Quality Checklist

- [ ] Are all JUnit 4 imports banned, using exclusively `org.junit.jupiter.api.*`?
- [ ] Are complex test suites structured using `@Nested` for readable BDD contexts?
- [ ] Are temporary files managed cleanly via `@TempDir` without hardcoded paths?
- [ ] Are domain entities and value objects tested with pure unit tests without any mocks?
- [ ] Are use cases tested with In-Memory Fakes rather than extensive Mockito `when(...)` stubs?
- [ ] Are Testcontainers shared via the Singleton Container Pattern instead of recreated per class?
- [ ] Is `@DirtiesContext` eliminated to preserve Spring's ApplicationContext cache?
- [ ] Are database tables/collections reset in `@BeforeEach`?
- [ ] Is `Thread.sleep()` strictly prohibited, using `Awaitility` instead for asynchronous assertions?
- [ ] Is `java.time.Clock` injected in all time-sensitive services and fixed in tests?
- [ ] Are REST endpoints tested with RestAssured, deserializing JSON into immutable `record`s?
- [ ] Are external HTTP calls stubbed with WireMock rather than mocking HTTP clients?
- [ ] Are architectural boundaries enforced through automated ArchUnit tests?
- [ ] Are test classes named strictly adhering to the `<TargetClass>Test` (unit) and `<TargetClass>IntegrationTest` (integration) conventions?
- [ ] Is mutation testing (Pitest) configured to verify test effectiveness?
