# Testing Taxonomy & Fakes over Mocks in Clean Architecture

A professional testing strategy separates tests by architectural layer and purpose, prioritizing fast, deterministic unit tests and real-behavior verification over brittle mock assertions.

---

## 1. Layered Testing Taxonomy

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        CLEAN ARCHITECTURE TESTING PYRAMID                              │
├───────────────────────────────┬───────────────────────────────────┬────────────────────┤
│ Architectural Layer           │ Primary Test Style                │ Tooling & Pattern  │
├───────────────────────────────┼───────────────────────────────────┼────────────────────┤
│ Domain Layer                  │ Pure Unit Tests (Zero Mocks)      │ JUnit 5, AssertJ   │
│ (Entities & Value Objects)    │ Execution: < 1ms                  │ Instancio          │
├───────────────────────────────┼───────────────────────────────────┼────────────────────┤
│ Application Layer             │ Behavior / Orchestration Tests    │ In-Memory Fakes    │
│ (Use Cases / Impl)            │ Execution: < 10ms                 │ JUnit 5, AssertJ   │
├───────────────────────────────┼───────────────────────────────────┼────────────────────┤
│ Infrastructure Layer          │ Real-Behavior Integration Tests   │ Testcontainers     │
│ (Adapters, Repositories, Web) │ Execution: Seconds                │ WireMock, RestAssured│
└───────────────────────────────┴───────────────────────────────────┴────────────────────┘
```

---

## 2. Domain Layer: Pure Unit Tests (Zero Mocks)

Domain entities and value objects encapsulate enterprise invariants and state mutations. They must be tested **without any mocking libraries**.

### Best Practices:
- Test business rules, calculations, boundary constraints, and state transitions directly.
- Tests execute in sub-milliseconds without initializing any framework or container context.

```java
class CustomerTest {

    @Test
    @DisplayName("Should successfully suspend active customer with valid reason")
    void shouldSuspendActiveCustomer() {
        var customer = new Customer(UUID.randomUUID(), "Alice", new Email("alice@example.com"));

        customer.suspend("Suspicious transaction pattern");

        assertThat(customer.getStatus()).isEqualTo(CustomerStatus.SUSPENDED);
    }

    @Test
    @DisplayName("Should throw exception when attempting to suspend already suspended customer")
    void shouldRejectSuspendingAlreadySuspendedCustomer() {
        var customer = new Customer(UUID.randomUUID(), "Alice", new Email("alice@example.com"));
        customer.suspend("First suspension");

        assertThatThrownBy(() -> customer.suspend("Second suspension"))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("already suspended");
    }
}
```

---

## 3. Application Layer: "Fakes & Stubs over Mocks"

### Why Heavy Mockito Usage is an Anti-Pattern in Use Cases:
- Tests coupled to `when(repo.findById(any())).thenReturn(...)` break whenever the internal implementation changes, even if external behavior remains identical (brittle tests).
- Mocking does not verify whether query filters, sorting, or state persistence actually behave correctly.
- Verbose mock setups obscure the actual business scenario being tested.

### The In-Memory Fake Pattern:
Because Clean Architecture defines Outbound Ports as clean Java interfaces, creating an **In-Memory Fake** backed by a `ConcurrentHashMap` or `List` is trivial and provides a high-fidelity test double.

#### Example: Outbound Port & In-Memory Fake Implementation
```java
// Outbound Port Interface (in Application Layer)
public interface CustomerRepositoryPort {
    void save(Customer customer);
    Optional<Customer> findById(UUID id);
    boolean existsByEmail(Email email);
}

// In-Memory Fake Test Double (in test package)
public final class InMemoryCustomerRepository implements CustomerRepositoryPort {
    private final Map<UUID, Customer> database = new ConcurrentHashMap<>();

    @Override
    public void save(Customer customer) {
        database.put(customer.getId(), customer);
    }

    @Override
    public Optional<Customer> findById(UUID id) {
        return Optional.ofNullable(database.get(id));
    }

    @Override
    public boolean existsByEmail(Email email) {
        return database.values().stream()
            .anyMatch(c -> c.getEmail().equals(email));
    }

    public void clear() {
        database.clear();
    }
}
```

#### Use Case Test Using the Fake:
```java
class RegisterCustomerUseCaseImplTest {

    private final InMemoryCustomerRepository customerRepository = new InMemoryCustomerRepository();
    private final FakeEventPublisher eventPublisher = new FakeEventPublisher();
    private RegisterCustomerUseCase useCase;

    @BeforeEach
    void setUp() {
        customerRepository.clear();
        eventPublisher.clear();
        useCase = new RegisterCustomerUseCaseImpl(customerRepository, eventPublisher);
    }

    @Test
    @DisplayName("Should register new customer and publish event")
    void shouldRegisterNewCustomerSuccessfully() {
        var command = new RegisterCustomerCommand("Bob", "bob@example.com");

        Result<CustomerRegisteredResponse, CustomerRegistrationError> result = useCase.execute(command);

        assertThat(result).isInstanceOf(Result.Success.class);
        assertThat(customerRepository.existsByEmail(new Email("bob@example.com"))).isTrue();
        assertThat(eventPublisher.getPublishedEvents()).hasSize(1);
    }
}
```

### When Mocks are Permitted:
Reserve mocks (Mockito) exclusively for third-party infrastructure boundaries (e.g., verifying that a credit bureau client was called with specific authentication headers) or hard-to-reproduce failure simulations (e.g. simulating low-level network socket exceptions).
