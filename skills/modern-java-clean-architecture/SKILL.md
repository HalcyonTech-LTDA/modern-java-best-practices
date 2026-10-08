---
name: modern-java-clean-architecture
description: >-
  Use this skill when architecting, implementing, or refactoring Java services using Screaming Architecture, Clean Architecture, OOP and Data-Oriented Programming (DOP):
  feature-first package structure, cross-feature communication, pure domain models, framework-free application layer, rich OOP entities,
  Value Objects with records, sealed domain event and error types (Result<T, E>), inbound/outbound ports (Ports & Adapters), and framework-agnostic interactors.
---

# Screaming Clean Architecture, OOP & DOP Synergy with Modern Java

This skill provides patterns for structuring enterprise Java applications cleanly (Modern Java Standard Baseline), organizing packages by business capability (**Screaming Architecture**), keeping core business and application layers framework-free (**Clean Architecture**), and solving **Cross-Feature Communication** without tight coupling.

## Screaming Package Structure (Feature-First)

```text
com.example.app/
├── ordering/                              # Feature 1 (Business Capability)
│   ├── application/                       # PURE JAVA
│   │   ├── api/                           # Inbound Ports (interfaces)
│   │   │   └── impl/                      # Inbound Port Implementations
│   │   ├── gateway/                       # Outbound Ports (interfaces)
│   │   └── usecase/                       # Core Use Cases
│   │       └── impl/                      # UseCase Implementations
│   ├── domain/                            # PURE JAVA: Heart of Business
│   │   ├── exception/                     # Domain-specific Exceptions
│   │   ├── model/                         # Entities, Value Objects, Events
│   │   └── repository/                    # Persistence Interfaces
│   └── infrastructure/                    # Adapters & Frameworks
│       ├── acl/                           # Anti-Corruption Layer
│       ├── config/                        # Spring Configs
│       ├── database/                      # DB Adapters, Entities, Repositories
│       ├── job/                           # Scheduled Jobs
│       ├── web/                           # Controllers, Request/Response DTOs
│       └── webclient/                     # HTTP Clients (Outbound)
│
├── billing/                               # Feature 2 (Business Capability)
│   ├── application/                       # API, Gateway, UseCase
│   ├── domain/                            # Model, Exception, Repository
│   └── infrastructure/                    # Database, Web, WebClient, Config
│
└── shared/                                # Minimal Shared Kernel (Primitives only)
    └── domain/                            # Universal Value Objects: Currency, TenantId
```

## Layer Independence Rule

Both the **Domain Layer** AND the **Application Layer** in each feature are 100% pure Java. They must never import, annotate with, or depend upon Spring Boot, Jakarta/JPA, Jackson, or any other framework.

## Key Topics & References

1. **Screaming Architecture & Cross-Feature Communication**: [Screaming Architecture & Cross-Feature Guide](./references/screaming_architecture_and_cross_feature.md)
   - Feature-first package layout shouting domain capabilities over framework technicalities.
   - Synchronous module communication via Outbound Port + ACL Adapter calling neighboring Inbound Ports.
   - Asynchronous decoupled communication via immutable Domain Events.
   - Minimal shared kernel boundaries.

2. **Domain Modeling (OOP Entities + DOP Value Objects)**: [Domain Models & Records](./references/domain_and_records.md)
   - Rich OOP domain entities with encapsulated state and invariants (avoiding anemic models).
   - Immutable Value Objects using records with compact constructor validation.
   - Sealed hierarchies for domain events and algebraic error types (`Result<T, E>`).

3. **Application Layer & Inbound/Outbound Ports**: [Use Cases & Ports](./references/use_cases_and_ports.md)
   - Defining inbound and outbound ports as clean Java interfaces with record commands/queries.
   - Pure Java interactors with constructor injection on `final` fields.
   - Transaction boundary management without framework coupling.

4. **Complete Framework Isolation**: [Framework Isolation Guide](./references/framework_isolation_guide.md)
   - Quarantining Spring to `infrastructure/config` via `@Configuration` bean factory methods.
   - Decoupling persistence models (JPA `@Entity` / Mongo `@Document`) from pure domain models.
   - Architecture validation rules for zero-framework leakage.

## Screaming & Clean Architecture Checklist

- [ ] Does the package structure scream business features at the root level (`ordering`, `billing`), not technical layers?
- [ ] Is cross-feature communication decoupled: zero direct imports of neighboring entities or database tables?
- [ ] Are synchronous cross-feature calls routed through Feature A's Outbound Port, implemented by an ACL adapter invoking Feature B's Inbound Port?
- [ ] Are side-effects across features handled asynchronously via Domain Events?
- [ ] Is the `domain` module strictly framework-free (no `@Component`, `@Entity`, `@JsonProperty`, or framework imports)?
- [ ] Is the `application` module strictly framework-free (no `@Service`, `@Component`, `@Transactional`, or `@Autowired`)?
- [ ] Are Domain Entities rich OOP classes with encapsulated state mutations, not anemic classes with getters/setters?
- [ ] Are Value Objects defined as immutable `record`s with invariant checks in compact constructors?
- [ ] Are domain errors represented using typed domain exceptions or sealed `Result<T, E>` types instead of generic `RuntimeException`s?
- [ ] Is dependency injection done exclusively via constructor injection on `final` fields?
