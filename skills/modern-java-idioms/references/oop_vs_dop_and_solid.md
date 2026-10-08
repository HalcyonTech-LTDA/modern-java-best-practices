# Object-Oriented Programming (OOP), SOLID & Data-Oriented Programming (DOP)

Modern Java is a multi-paradigm language. Rather than treating **Object-Oriented Programming (OOP)** and **Data-Oriented Programming (DOP)** as mutually exclusive, modern Java blends them into a pragmatic, robust architecture.

---

## 1. Core OOP Principles in Modern Java

Classic OOP remains the foundational paradigm for modeling identity, lifecycle, behavioral abstraction, and state encapsulation.

### 1. Encapsulation & Information Hiding
- **Protecting Invariants**: An entity with an identity and lifecycle must guard its internal state against invalid transitions. State mutations happen through expressive business methods, never via naked setters.
- **Example (Rich OOP Entity)**:
  ```java
  public class BankAccount {
      private final AccountId id;
      private Money balance;
      private AccountStatus status;

      public BankAccount(AccountId id, Money initialDeposit) {
          this.id = Objects.requireNonNull(id, "Account ID is required");
          this.balance = Objects.requireNonNull(initialDeposit, "Initial deposit required");
          this.status = AccountStatus.ACTIVE;
      }

      // Encapsulated behavior protecting business rules
      public void withdraw(Money amount) {
          if (this.status != AccountStatus.ACTIVE) {
              throw new IllegalStateException("Cannot withdraw from inactive account");
          }
          if (this.balance.isLessThan(amount)) {
              throw new InsufficientFundsException("Balance %s is insufficient for withdrawal %s".formatted(balance, amount));
          }
          this.balance = this.balance.subtract(amount);
      }

      public AccountId getId() { return id; }
      public Money getBalance() { return balance; }
      public AccountStatus getStatus() { return status; }
  }
  ```

### 2. Polymorphism (Open Dispatch)
- Use standard Java `interface` contracts when external or third-party implementations can be added without modifying existing code (e.g., adding a new payment gateway or database adapter).

### 3. Composition over Inheritance
- Favor composing small, focused classes and delegating behavior over deep class inheritance hierarchies. Class inheritance should be reserved for strict "is-a" relationships or sealed hierarchies.

---

## 2. SOLID Principles in Modern Java

Applying SOLID in modern Java eliminates enterprise bloat while preserving long-term maintainability:

| Principle | Classic Definition | Modern Java Application |
| :--- | :--- | :--- |
| **S - Single Responsibility** | A class should have one reason to change. | Separate transparent data holders (`record`), rich stateful entities (OOP `class`), and orchestration workflows (`Use Case`). |
| **O - Open/Closed** | Open for extension, closed for modification. | Use standard `interface` for open extension (adapters, plugins). Use `sealed interface` for closed domain variants where the compiler enforces exhaustive pattern matching. |
| **L - Liskov Substitution** | Subtypes must be substitutable for base types. | Implementations of outbound ports (SPIs) must satisfy the contract without throwing unexpected runtime exceptions. Sealed permits must strictly honor the root contract. |
| **I - Interface Segregation** | Clients should not depend on methods they do not use. | Define granular, use-case-specific Inbound Ports (`RegisterCustomerUseCase`, `ChangePasswordUseCase`) instead of giant monolithic services. |
| **D - Dependency Inversion** | High-level modules depend on abstractions. | Use case implementations depend on pure Java port interfaces. Infrastructure adapters implement those interfaces. |

---

## 3. Decision Matrix: When to Use OOP vs. DOP vs. Hybrid

Choosing between OOP and DOP depends on the nature of the problem, the data lifecycle, and how behavior evolves.

```text
                                  Does the concept have a
                                distinct identity and lifecycle
                                   over time (mutable state)?
                                          /          \
                                        YES           NO
                                        /              \
                                   [USE OOP]      Is the set of types
                           (Rich Entities/Aggregates)  closed and known at
                                                       compile time?
                                                        /         \
                                                      YES          NO
                                                      /             \
                                                  [USE DOP]      [USE OOP]
                                              (Sealed Sum Types) (Standard Interface)
```

### Detailed Comparison Table

| Dimension | Classic OOP | Data-Oriented Programming (DOP) | Pragmatic Hybrid (Clean Architecture) |
| :--- | :--- | :--- | :--- |
| **Primary Goal** | Encapsulate state + behavior together | Model transparent data separately from logic | OOP for orchestration; DOP for data |
| **Identity** | Significant (UUID, database ID, memory address) | Meaningless (equality based strictly on values) | Entities have identity; Value Objects do not |
| **Mutability** | Controlled, encapsulated mutability | Pure immutability (`record`) | Entities mutate safely; data carriers are immutable |
| **Extensibility (Expression Problem)** | Easy to add **new types**; hard to add **new operations** | Easy to add **new operations**; hard to add **new types** | Best of both: open ports for new plugins; sealed types for domain rules |
| **Where to Use** | Aggregates, Entities, Ports, Use Cases, Adapters | Value Objects, DTOs, Commands, Queries, Events, Results | Complete application architecture |

---

## 4. The Pragmatic Hybrid Architecture in Practice

In an enterprise Clean Architecture service, OOP and DOP work hand-in-hand:

### Layer-by-Layer Synergy:
1. **Inbound & Outbound Ports (OOP)**:
   - Interfaces enforce DIP and ISP.
   - Example: `PlaceOrderUseCase` and `OrderRepositoryPort`.
2. **Commands, Queries & DTOs (DOP)**:
   - Input and output data carriers are immutable `record`s.
   - Example: `PlaceOrderCommand(UUID customerId, List<OrderItemDto> items)`.
3. **Use Case Implementation (OOP)**:
   - Stateless service implementing SRP, orchestrating the interaction between domain entities and ports.
4. **Aggregate Root & Domain Entity (OOP)**:
   - Rich class encapsulating invariants and executing state transitions (`order.addItem(...)`, `order.confirmPayment(...)`).
5. **Value Objects (DOP)**:
   - Nested within entities as immutable `record`s with compact constructors (`Money`, `Email`, `Quantity`).
6. **Domain Events & Algebraic Errors (DOP)**:
   - Emitted by aggregates and interactors as `sealed interface` sum types, processed via exhaustive `switch` pattern matching.

### End-to-End Hybrid Flow Example:
```java
// 1. DOP: Command Record
public record PlaceOrderCommand(UUID customerId, List<ItemOrderDto> items) {}

// 2. OOP: Inbound Port Interface
public interface PlaceOrderUseCase {
    Result<OrderConfirmedResponse, OrderError> execute(PlaceOrderCommand command);
}

// 3. OOP + DOP: Rich Domain Entity (OOP) containing Value Objects (DOP)
public class Order {
    private final OrderId id; // DOP Value Object
    private final CustomerId customerId; // DOP Value Object
    private final List<OrderItem> items = new ArrayList<>();
    private OrderStatus status; // Enum / Sealed

    public Order(OrderId id, CustomerId customerId) {
        this.id = Objects.requireNonNull(id);
        this.customerId = Objects.requireNonNull(customerId);
        this.status = OrderStatus.PENDING;
    }

    // OOP Encapsulated state transition
    public void addLineItem(ProductId product, Quantity qty, Money unitPrice) {
        if (this.status != OrderStatus.PENDING) {
            throw new IllegalStateException("Cannot add items to non-pending order");
        }
        this.items.add(new OrderItem(product, qty, unitPrice));
    }
}

// 4. OOP: Use Case Implementation orchestrating the workflow
public final class PlaceOrderUseCaseImpl implements PlaceOrderUseCase {
    private final OrderRepositoryPort orderRepository; // OOP Port

    public PlaceOrderUseCaseImpl(OrderRepositoryPort orderRepository) {
        this.orderRepository = Objects.requireNonNull(orderRepository);
    }

    @Override
    public Result<OrderConfirmedResponse, OrderError> execute(PlaceOrderCommand command) {
        Order order = new Order(OrderId.random(), new CustomerId(command.customerId()));
        
        for (var item : command.items()) {
            order.addLineItem(new ProductId(item.productId()), new Quantity(item.qty()), new Money(item.price()));
        }

        orderRepository.save(order);
        return Result.success(new OrderConfirmedResponse(order.getId().value()));
    }
}
```
