# Data-Oriented Programming (DOP) in Modern Java

Data-Oriented Programming (DOP) is a paradigm that models data as first-class, immutable citizens, strictly separated from behavior. Modern Java provides first-class language support for DOP through the combination of **Records**, **Sealed Types**, and **Pattern Matching**.

---

## 1. The Four Principles of DOP

As formalized by Brian Goetz, DOP in Java rests on four core principles:

1. **Model Data Separately from Behavior**: Data entities are transparent, shallowly immutable carriers (`record`). Business rules and domain workflows live in stateless domain services, interactors, and pure functions.
2. **Model Data Variants as Closed Algebraic Types (Sum Types)**: Use `sealed interface` or `sealed class` hierarchies to represent all valid possibilities of a domain concept.
3. **Data is Immutable by Default**: Data cannot be modified after construction. Transformations create new instances rather than mutating state in place.
4. **Process Data via Pattern Matching & Deconstruction**: Extract values and branch on data structures using `switch` expressions and record patterns directly, without calling getter methods or performing manual type casting.

---

## 2. Comparing Traditional OOP vs. Modern DOP

| Aspect | Traditional OOP | Modern Java DOP |
| :--- | :--- | :--- |
| **Data Representation** | Mutable classes with private fields, getters, setters | Transparent, immutable `record`s |
| **Type Hierarchies** | Open class inheritance (`extends BaseEntity`) | Closed algebraic sum types (`sealed interface permits A, B, C`) |
| **Logic Placement** | Methods embedded inside mutable objects | Pure functions and domain services operating on immutable data |
| **State Inspection** | Calling getters (`item.getDetails().getPrice()`) | Direct pattern deconstruction (`case Item(Details(_, var price))`) |
| **Compiler Verification** | Runtime `NullPointerException` or unhandled cases | Compile-time exhaustiveness checking without `default` |

---

## 3. Modeling Algebraic Data Types (Sum Types & Product Types)

- **Product Types**: Combine multiple attributes together into a single record (`Record Point(int x, int y)`).
- **Sum Types**: Represent an "OR" choice between a fixed set of possibilities (`sealed interface`).

### Example: Modeling E-Commerce Discount Policies
```java
package br.com.example.domain.model;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.Objects;

public sealed interface DiscountPolicy permits 
        DiscountPolicy.NoDiscount, 
        DiscountPolicy.PercentageOff, 
        DiscountPolicy.FixedAmount, 
        DiscountPolicy.SeasonalPromotion {

    record NoDiscount() implements DiscountPolicy {}

    record PercentageOff(BigDecimal percentage) implements DiscountPolicy {
        public PercentageOff {
            Objects.requireNonNull(percentage, "Percentage must not be null");
            if (percentage.compareTo(BigDecimal.ZERO) <= 0 || percentage.compareTo(BigDecimal.valueOf(100)) > 0) {
                throw new IllegalArgumentException("Percentage must be between 0 and 100");
            }
        }
    }

    record FixedAmount(BigDecimal amount) implements DiscountPolicy {
        public FixedAmount {
            Objects.requireNonNull(amount, "Amount must not be null");
            if (amount.compareTo(BigDecimal.ZERO) <= 0) {
                throw new IllegalArgumentException("Amount must be positive");
            }
        }
    }

    record SeasonalPromotion(String promoCode, BigDecimal percentage, LocalDate validUntil) implements DiscountPolicy {
        public SeasonalPromotion {
            Objects.requireNonNull(promoCode, "Promo code cannot be null");
            Objects.requireNonNull(percentage, "Percentage cannot be null");
            Objects.requireNonNull(validUntil, "Expiry date cannot be null");
        }
    }

    record BundlePromotion(List<String> requiredSkus, BigDecimal percentage) implements DiscountPolicy {
        public BundlePromotion {
            Objects.requireNonNull(requiredSkus, "requiredSkus cannot be null");
            Objects.requireNonNull(percentage, "percentage cannot be null");
            requiredSkus = List.copyOf(requiredSkus); // Defensive copy: ensures deep immutability
        }
    }
}
```

---

## 4. Processing Data via Exhaustive Pattern Matching

Because `DiscountPolicy` is a sealed interface, the compiler enforces exhaustiveness. If a developer later introduces a new permit, every switch expression without that permit will immediately fail at compile time.

```java
package br.com.example.domain.service;

import br.com.example.domain.model.DiscountPolicy;
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDate;

public final class PricingService {

    public BigDecimal calculateEffectivePrice(BigDecimal basePrice, DiscountPolicy policy, LocalDate orderDate) {
        return switch (policy) {
            case DiscountPolicy.NoDiscount() -> 
                basePrice;

            case DiscountPolicy.FixedAmount(var amount) -> 
                basePrice.subtract(amount).max(BigDecimal.ZERO);

            case DiscountPolicy.PercentageOff(var pct) -> {
                BigDecimal discount = basePrice.multiply(pct).divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_UP);
                yield basePrice.subtract(discount).max(BigDecimal.ZERO);
            }

            case DiscountPolicy.SeasonalPromotion(var code, var pct, var expiry) when orderDate.isBefore(expiry.plusDays(1)) -> {
                BigDecimal discount = basePrice.multiply(pct).divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_UP);
                yield basePrice.subtract(discount).max(BigDecimal.ZERO);
            }

            case DiscountPolicy.SeasonalPromotion(var code, _, _) -> 
                basePrice; // Promotion expired
        };
    }
}
```

Notice:
1. No getters like `policy.getPercentage()` are used; values are bound directly via record patterns.
2. The `when` guard cleanly isolates the expiration rule without nested `if` statements.
3. No redundant `default:` case is needed.
