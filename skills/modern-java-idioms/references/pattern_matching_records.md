# Pattern Matching & Records in Modern Java

Modern Java elevates data-oriented programming by pairing **Records**, **Sealed Types**, **Pattern Matching**, and **Unnamed Variables**.

---

## 1. Sealed Type Hierarchies & Algebraic Data Types

Model domain variants using a `sealed` interface with record implementations:

```java
public sealed interface PaymentMethod permits CreditCard, Pix, BankTransfer {
}

public record CreditCard(String cardNumber, String holderName, YearMonth expiry) implements PaymentMethod {}
public record Pix(String pixKey, PixKeyType keyType) implements PaymentMethod {}
public record BankTransfer(String routingNumber, String accountNumber) implements PaymentMethod {}
```

### Exhaustive Switch Without `default`
Because the compiler knows all permitted subtypes, omitting `default` ensures a compile-time failure if a new subtype is added in the future:

```java
public String processPayment(PaymentMethod method) {
    return switch (method) {
        case CreditCard(var number, var holder, _) -> 
            "Processing CC ending in " + number.substring(number.length() - 4);
        case Pix(var key, var type) -> 
            "Processing Pix of type " + type + " with key: " + key;
        case BankTransfer(var routing, var account) -> 
            "Transferring to account: " + account + " (Routing: " + routing + ")";
    };
}
```

---

## 2. Nested Record Patterns & Deconstruction

You can deconstruct nested records directly inside pattern matching:

```java
public record Address(String street, String city, String zipCode) {}
public record Customer(UUID id, String name, Address address) {}
public record Order(UUID orderId, Customer customer, BigDecimal total) {}

public String formatDeliveryLabel(Order order) {
    if (order instanceof Order(_, Customer(var name, Address(var street, var city, var zip)), _)) {
        return "%s\n%s\n%s, %s".formatted(name, street, city, zip);
    }
    return "Unknown delivery address";
}
```

---

## 3. Pattern Matching with `when` Guards

Guards refine pattern matches with boolean expressions:

```java
public String categorizeTransaction(Transaction tx) {
    return switch (tx) {
        case Transaction(var id, var amount, _) when amount.compareTo(BigDecimal.valueOf(10_000)) >= 0 ->
            "High-value audit required: " + id;
        case Transaction(var id, var amount, _) when amount.compareTo(BigDecimal.ZERO) < 0 ->
            "Refund or chargeback: " + id;
        case Transaction(var id, _, _) ->
            "Standard transaction: " + id;
    };
}
```

---

## 4. Unnamed Variables & Patterns (`_`)

The underscore `_` denotes variables or components that are intentionally ignored:

### Catch Clauses
```java
public Optional<Integer> parseSafe(String input) {
    try {
        return Optional.of(Integer.parseInt(input));
    } catch (NumberFormatException _) {
        return Optional.empty();
    }
}
```

### Unused Lambda Parameters
```java
map.computeIfAbsent(key, _ -> new ArrayList<>());
```

### Unused Pattern Components
```java
if (event instanceof UserRegisteredEvent(var userId, _, _)) {
    auditLogger.info("User registered: {}", userId);
}
```

---

## 5. Local Records for Scoped Intermediate Data

Declare local `record`s inside methods to encapsulate intermediate structured queries, calculation tuples, or grouping keys without polluting the enclosing class or package namespace:

```java
public Map<CustomerId, BigDecimal> calculateCustomerTotals(List<Order> orders) {
    record CustomerSubTotal(CustomerId customerId, BigDecimal total) {}

    return orders.stream()
        .map(order -> new CustomerSubTotal(order.getCustomerId(), order.getTotal()))
        .collect(Collectors.groupingBy(
            CustomerSubTotal::customerId,
            Collectors.reducing(BigDecimal.ZERO, CustomerSubTotal::total, BigDecimal::add)
        ));
}
```
