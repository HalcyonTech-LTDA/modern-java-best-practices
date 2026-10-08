# Domain Modeling: OOP Entities & DOP Value Objects

The domain layer encapsulates enterprise business rules and must remain 100% independent of frameworks, databases, and network transports. 

Modern Java Clean Architecture leverages a **Pragmatic Hybrid Approach**:
- **Classic OOP for Domain Entities & Aggregate Roots**: Modeling identity, lifecycle, state mutations, and invariant encapsulation.
- **Data-Oriented Programming (DOP) for Value Objects, Events & Errors**: Modeling immutable data, algebraic sum types, and pattern-based evaluation.

---

## 1. Value Objects as Records (DOP)

Value Objects represent descriptive aspects of the domain with no conceptual identity. Two value objects with identical fields are considered equal. Modern Java `record`s are the ideal vehicle because they are deeply immutable and transparent:

```java
package br.com.example.domain.model;

import java.util.Objects;
import java.util.regex.Pattern;

public record Email(String value) {
    private static final Pattern EMAIL_REGEX = 
        Pattern.compile("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+$");

    public Email {
        Objects.requireNonNull(value, "Email value cannot be null");
        String normalized = value.strip().toLowerCase();
        if (!EMAIL_REGEX.matcher(normalized).matches()) {
            throw new IllegalArgumentException("Invalid email format: " + value);
        }
        value = normalized; // compact constructor normalization
    }
}
```

```java
package br.com.example.domain.model;

import java.math.BigDecimal;
import java.util.Currency;
import java.util.Objects;

public record Money(BigDecimal amount, Currency currency) {
    public Money {
        Objects.requireNonNull(amount, "Amount cannot be null");
        Objects.requireNonNull(currency, "Currency cannot be null");
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency())) {
            throw new IllegalArgumentException("Currency mismatch: %s vs %s".formatted(currency, other.currency()));
        }
        return new Money(this.amount.add(other.amount()), this.currency);
    }
}
```java
public record OrderPlacedEvent(UUID orderId, List<OrderItem> items) {
    public OrderPlacedEvent {
        Objects.requireNonNull(orderId, "orderId is required");
        items = List.copyOf(items); // Enforces deep immutability against external caller mutation
    }
}
```

### Defensive Copies in Compact Constructors
Records provide shallow immutability. When a record encapsulates a collection or mutable structure, always perform a defensive copy in its compact constructor using `List.copyOf()`, `Set.copyOf()`, or `Map.copyOf()`.

---

## 2. Rich Domain Entities & Aggregates (Classic OOP)

Unlike Value Objects, Domain Entities have a persistent **identity** that spans time and state mutations. 

### Core OOP Principles Applied to Entities:
1. **Encapsulation & Information Hiding**: Fields are `private`. State can only be altered through domain methods that enforce business invariants.
2. **Tell, Don't Ask**: Do not query an entity's fields to perform business calculations outside; invoke a method on the entity so it executes its own logic.
3. **No Anemic Domain Models**: Prohibit naked setters (`setName()`, `setStatus()`). Use intent-revealing business methods (`activate()`, `suspend()`, `changeEmail()`).

```java
package br.com.example.domain.model;

import java.util.Objects;
import java.util.UUID;

// ZERO framework annotations (@Entity, @Table, @Id from JPA are strictly prohibited here)
public class Customer {
    private final UUID id;             // Identity (OOP)
    private String name;               // Mutable state
    private Email email;               // DOP Value Object
    private CustomerStatus status;     // Encapsulated state variant

    public Customer(UUID id, String name, Email email) {
        this.id = Objects.requireNonNull(id, "ID cannot be null");
        this.name = Objects.requireNonNull(name, "Name cannot be null");
        this.email = Objects.requireNonNull(email, "Email cannot be null");
        this.status = CustomerStatus.ACTIVE;
    }

    // OOP Encapsulated Business Method enforcing Invariants
    public void suspend(String reason) {
        if (this.status == CustomerStatus.SUSPENDED) {
            throw new IllegalStateException("Customer is already suspended");
        }
        if (reason == null || reason.isBlank()) {
            throw new IllegalArgumentException("Suspension reason cannot be empty");
        }
        this.status = CustomerStatus.SUSPENDED;
    }

    public void updateContactEmail(Email newEmail) {
        if (this.status == CustomerStatus.SUSPENDED) {
            throw new IllegalStateException("Cannot update email of a suspended customer");
        }
        this.email = Objects.requireNonNull(newEmail, "New email cannot be null");
    }

    // Read-only getters exposing immutable or primitive data
    public UUID getId() { return id; }
    public String getName() { return name; }
    public Email getEmail() { return email; }
    public CustomerStatus getStatus() { return status; }
}
```

---

## 3. Algebraic Domain Results (`Result<T, E>`) & Events (DOP)

Using a `sealed` interface for operation results models expected business outcomes as closed sum types without runtime exception overhead:

```java
package br.com.example.domain.model;

public sealed interface Result<T, E> {
    record Success<T, E>(T value) implements Result<T, E> {}
    record Failure<T, E>(E error) implements Result<T, E> {}

    static <T, E> Result<T, E> success(T value) {
        return new Success<>(value);
    }

    static <T, E> Result<T, E> failure(E error) {
        return new Failure<>(error);
    }
}
```

### Domain Errors as Closed Sealed Hierarchies:
```java
package br.com.example.domain.model;

public sealed interface OrderError {
    record InsufficientInventory(String productId, int requested, int available) implements OrderError {}
    record CustomerSuspended(String customerId, String reason) implements OrderError {}
    record PaymentGatewayUnavailable(String provider) implements OrderError {}
}
```

### Consuming Results via Exhaustive Switch (Infrastructure Layer)
Pattern matching guarantees compile-time verification that every domain error branch is handled:

```java
// Inside an Infrastructure Web Controller / Presenter:
Result<Order, OrderError> outcome = placeOrderUseCase.execute(command);

return switch (outcome) {
    case Result.Success(var order) -> 
        HttpResponse.ok(OrderDto.from(order));
    case Result.Failure(OrderError.InsufficientInventory(_, var req, var avail)) ->
        HttpResponse.badRequest("Insufficient inventory: requested %d, available %d".formatted(req, avail));
    case Result.Failure(OrderError.CustomerSuspended(_, var reason)) ->
        HttpResponse.forbidden("Customer suspended: " + reason);
    case Result.Failure(OrderError.PaymentGatewayUnavailable(var provider)) ->
        HttpResponse.serviceUnavailable("Payment provider unavailable: " + provider);
};
```
