---
name: modern-java-idioms
description: >-
  Use this skill when writing, modernizing, or refactoring Java code to utilize modern Java language features (Modern Java Standard Baseline):
  Object-Oriented Programming (OOP) & SOLID principles, Data-Oriented Programming (DOP), Records and Record Patterns,
  exhaustive switch expressions, Sealed Hierarchies, Unnamed Variables and Patterns, Sequenced Collections,
  Flexible Constructor Bodies, Stream Gatherers, Dependency Hygiene (Anti-Lombok), and the Foreign Function & Memory (FFM) API.
---

# Modern Java Language Idioms, OOP & Data-Oriented Programming

This skill guides you through applying modern language enhancements, Object-Oriented Programming (OOP) and SOLID best practices, Data-Oriented Programming (DOP), and standard library idioms (Modern Java Standard Baseline).

## Key Capabilities & Topic Guides

1. **OOP, SOLID Principles & Paradigm Decision Matrix**:
   - Apply SOLID in modern Java: Single Responsibility, Open/Closed (interfaces vs. sealed sum types), Liskov Substitution, Interface Segregation (narrow use case ports), and Dependency Inversion.
   - Master the decision matrix: When to use classic OOP (rich entities, state encapsulation), when to use DOP (value objects, events, algebraic types), and how to build pragmatic hybrid architectures.
   - See comprehensive guide: [OOP, SOLID & DOP Guide](./references/oop_vs_dop_and_solid.md).

2. **Data-Oriented Programming (DOP)**:
   - Apply the 4 pillars of DOP: model data as transparent records, model states as sealed algebraic sum types, enforce immutability, and process via exhaustive pattern matching.
   - See detailed principles and examples: [Data-Oriented Programming Guide](./references/data_oriented_programming.md).

3. **Record Patterns & Exhaustive Switch**:
   - Deconstruct complex record hierarchies in `instanceof` and `switch`.
   - Apply pattern guards with `when` without nested `if/else`.
   - See pattern matching recipes: [Pattern Matching & Records](./references/pattern_matching_records.md).

4. **Dependency Hygiene & Code Style (Anti-Lombok)**:
   - Eliminate Lombok annotations (`@Data`, `@Getter`, `@Setter`, `@Builder`) in favor of native `record`s.
   - Enforce clean `var` usage, explicit top-level `import` declarations over inline FQCNs/PQCNs, `Stream.toList()`, Text Blocks (`"""`), and `String.formatted()`.
   - See full guidelines: [Dependency Hygiene & Style Guide](./references/dependency_hygiene_and_style.md).

5. **Stream Gatherers (`Stream.gather`)**:
   - Transform and manipulate stream pipelines using built-in gatherers (`windowFixed`, `windowSliding`, `fold`, `scan`, `mapConcurrent`).
   - Implement custom intermediate stream gatherers for domain-specific transformations.
   - See full guide: [Stream Gatherers Guide](./references/stream_gatherers.md).

6. **Sequenced Collections & Flexible Constructors**:
   - Standardize first/last and reverse collection operations with `SequencedCollection` and `SequencedMap`.
   - Validate and prepare constructor state prior to `super(...)` calls using flexible constructor bodies.
   - See complete reference: [Sequenced Collections & Constructors](./references/sequenced_collections_and_constructors.md).

7. **FFM API, Buffers & Modular Encapsulation (JPMS)**:
   - Replace legacy JNI and `sun.misc.Unsafe` with the Foreign Function & Memory API (`java.lang.foreign.*`).
   - Optimize high-throughput streaming with direct `ByteBuffer` and zero-copy `FileChannel`.
   - Enforce strict encapsulation in `module-info.java` and avoid reflection via `MethodHandles`.
   - See advanced guide: [FFM API & Buffers](./references/ffm_and_buffers.md).

## Quick Checklist for Code Modernization

- [ ] Is the paradigm choice deliberate: OOP for stateful entities/ports, DOP for transparent data/events/errors?
- [ ] Are SOLID principles respected: Single Responsibility, narrow Ports (ISP), Dependency Inversion (DIP)?
- [ ] Has all Lombok boilerplate (`@Data`, `@Getter`, `@Setter`, `@AllArgsConstructor`, `@Builder`) been eliminated?
- [ ] Are explicit top-level `import` declarations used instead of inline FQCN/package-qualified class names?
- [ ] Are record invariants and null assertions enforced inside a compact constructor?
- [ ] Have traditional `instanceof` + cast chains been refactored to record/type pattern matching?
- [ ] Are switches over sealed types exhaustive, without a redundant `default` branch?
- [ ] Are unused variables (in catches, lambda parameters, or pattern components) named `_`?
- [ ] Is `var` used only when the concrete type is obvious from the right-hand side?
- [ ] Is `Stream.toList()` used instead of `Collectors.toList()`.
- [ ] Are multiline strings formatted with Text Blocks (`"""`) and `String.formatted()`?
- [ ] Are sequenced collection methods (`getFirst()`, `getLast()`, `reversed()`) used instead of index math?
