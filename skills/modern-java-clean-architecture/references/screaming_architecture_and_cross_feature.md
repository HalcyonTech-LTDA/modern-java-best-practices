# Screaming Architecture & Cross-Feature Communication

Clean Architecture defines the concentric dependency rules between layers, but it does not specify how the root codebase should be organized. **Screaming Architecture** (first coined by Robert C. Martin) addresses this: your project's top-level structure should scream its business purpose (e.g., `ordering`, `billing`, `inventory`), rather than screaming technical stereotypes (`controllers`, `services`, `repositories`).

When combined, **Screaming Clean Architecture** organizes code by business feature first, and by Clean Architecture layers internally.

---

## 1. Package Structure: Feature-First (Screaming Clean Architecture)

### ❌ Anti-Pattern: Package-by-Layer (Technical Silos)
```text
com.example.app/
├── controller/       # All REST controllers dumped together
├── service/          # Every business service in the application
├── repository/       # All database repositories
└── model/            # All entities and DTOs mixed together
```
*Problems*: Everything must be `public`. Features are scattered across layers. Changing an "Order" requires touching four distant folders.

### ✅ Screaming Clean Architecture (Vertical Features with Pure Internal Layers)
```text
com.example.app/
├── ordering/                              # Feature 1 (Business Capability)
│   ├── domain/                            # 100% Pure Java: Order, OrderLine, OrderStatus
│   ├── application/                       # 100% Pure Java
│   │   ├── api/                           # Public API + Command/Query Records
│   │   │   └── impl/                      # API implementations
│   │   ├── gateway/                       # Outbound Gateways
│   │   └── usecase/                       # Core Use Cases
│   │       └── impl/                      # UseCase implementations
│   └── infrastructure/                    # Adapters, JPA Entities, Controllers, Spring Configs
│       ├── acl/                           # BillingModuleAdapter (ACL)
│       ├── database/adapter/              # OrderRepositoryAdapter
│       ├── web/                           # OrderRestController
│       └── config/                        # OrderingBeansConfiguration
│
├── billing/                               # Feature 2 (Business Capability)
│   ├── domain/                            # 100% Pure Java: Invoice, PaymentReceipt, Money
│   ├── application/                       # 100% Pure Java: ProcessPaymentUseCase, Inbound Ports
│   └── infrastructure/                    # Adapters, StripeClient, Spring Configs
│
└── shared/                                # Minimal Shared Kernel (Primitives only)
    └── domain/                            # Common Value Objects: Currency, TenantId
```

---

## 2. The Cross-Feature Communication Challenge

In a multi-feature system, features cannot exist in total isolation. An `ordering` use case inevitably needs to charge a customer via `billing` or reserve stock in `inventory`.

### 🚫 The 3 Cross-Feature Anti-Patterns
1. **Direct Entity Leaks**: `ordering` imports `billing.domain.Invoice`. This tightly couples domain models and breaks bounded contexts.
2. **Cross-Feature Database Joins**: `ordering` queries `billing` tables directly via SQL or JPA entity relationships (`@ManyToOne`).
3. **Circular Dependencies**: `ordering` calls `billing`, and `billing` calls `ordering`, making modular refactoring or extraction to microservices impossible.

---

## 3. Resolving Cross-Feature Communication

There are two primary, decoupled architectural patterns to solve cross-feature communication cleanly:

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                 ORDERING MODULE                                        │
│                                                                                        │
│   [Ordering UseCaseImpl]                                                               │
│            │                                                                           │
│            ▼ (depends on)                                                              │
│   [BillingGateway (Outbound Port)]                                                     │
│            ▲                                                                           │
│            │ (implements)                                                              │
│   [BillingModuleAdapter (ACL)] ───► Calls Public Inbound Port ──► [Billing UseCaseImpl]│
│      (in ordering/infrastructure)     (in billing/application/api)                     │
│                                                                                        │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

### Pattern 1: Synchronous In-Process Calls via Outbound Port + Adapter (ACL)

When Feature A needs an immediate response from Feature B:

1. **Feature A defines an Outbound Port** in its own `application/gateway`.
2. **Feature B exposes a Public Inbound Port** in its `application/api` (with public command/response records).
3. **Feature A's Infrastructure implements the Outbound Port** via an adapter that calls Feature B's Inbound Port. The adapter acts as an **Anti-Corruption Layer (ACL)**, mapping Feature A's concepts to Feature B's contract.

#### Step 1: Feature A (`ordering`) declares its need via an Outbound Port
```java
package com.example.app.ordering.application.gateway;

import com.example.app.ordering.domain.model.Money;
import com.example.app.ordering.domain.model.OrderId;
import com.example.app.ordering.domain.model.Result;

public interface BillingGateway {
    sealed interface BillingGatewayError {
        record PaymentDeclined(String reason) implements BillingGatewayError {}
        record ProviderUnavailable() implements BillingGatewayError {}
    }

    record PaymentConfirmation(String transactionId) {}

    Result<PaymentConfirmation, BillingGatewayError> requestPayment(OrderId orderId, Money amount, String paymentToken);
}
```

#### Step 2: Feature B (`billing`) exposes its Inbound Port (Public API)
```java
package com.example.app.billing.application.api;

import com.example.app.billing.domain.model.Result;
import java.math.BigDecimal;
import java.util.UUID;

public interface ProcessPaymentApi {
    record ProcessPaymentCommand(UUID referenceId, BigDecimal amount, String currency, String token) {}
    record PaymentProcessedResponse(String transactionId) {}
    
    sealed interface ProcessPaymentError {
        record Declined(String reason) implements ProcessPaymentError {}
        record Failure() implements ProcessPaymentError {}
    }

    Result<PaymentProcessedResponse, ProcessPaymentError> execute(ProcessPaymentCommand command);
}
```

#### Step 3: Feature A's Infrastructure Adapter bridges the two modules (ACL)
```java
package com.example.app.ordering.infrastructure.acl;

import com.example.app.billing.application.api.ProcessPaymentApi;
import com.example.app.billing.application.api.ProcessPaymentApi.ProcessPaymentCommand;
import com.example.app.billing.application.api.ProcessPaymentApi.ProcessPaymentError;
import com.example.app.ordering.application.gateway.BillingGateway;
import com.example.app.ordering.domain.model.Money;
import com.example.app.ordering.domain.model.OrderId;
import com.example.app.ordering.domain.model.Result;
import org.springframework.stereotype.Component;

@Component
public class BillingModuleAdapter implements BillingGateway {

    private final ProcessPaymentApi processPaymentApi; // Injected Inbound Port from Billing

    public BillingModuleAdapter(ProcessPaymentApi processPaymentApi) {
        this.processPaymentApi = processPaymentApi;
    }

    @Override
    public Result<PaymentConfirmation, BillingGatewayError> requestPayment(OrderId orderId, Money amount, String token) {
        // ACL: Translate Ordering models -> Billing command
        var command = new ProcessPaymentCommand(
            orderId.value(),
            amount.amount(),
            amount.currency().getCurrencyCode(),
            token
        );

        var billingResult = processPaymentUseCase.execute(command);

        // ACL: Translate Billing response/error -> Ordering domain result
        return switch (billingResult) {
            case Result.Success(var resp) -> 
                Result.success(new PaymentConfirmation(resp.transactionId()));
            case Result.Failure(ProcessPaymentError.Declined(var reason)) -> 
                Result.failure(new BillingGatewayError.PaymentDeclined(reason));
            case Result.Failure(ProcessPaymentError.Failure()) -> 
                Result.failure(new BillingGatewayError.ProviderUnavailable());
        };
    }
}
```

*Key Benefit*: The `ordering` domain and application layers have **zero imports** from `billing`. If `billing` changes its internal database, domain models, or is extracted into an independent microservice, only `BillingModuleAdapter` changes.

---

### Pattern 2: Asynchronous Decoupling via Domain Events

When Feature A does not need a synchronous response from Feature B (fire-and-forget or eventual consistency):

1. **Feature A publishes a Domain Event** upon completing its use case (`OrderPlacedEvent`).
2. **Feature B listens to the event** via an infrastructure event listener and executes its own use case (`GenerateInvoiceUseCase`).
3. **Zero Direct Dependency**: Feature A does not even know Feature B exists.

```text
┌────────────────────────────────┐         ┌────────────────────────────────┐
│        ORDERING MODULE         │         │         BILLING MODULE         │
│                                │         │                                │
│   [PlaceOrderUseCaseImpl]      │         │   [OrderPlacedEventListener]   │
│            │                   │         │            │                   │
│            ▼                   │         │            ▼                   │
│   Publishes OrderPlacedEvent   │ ──────► │   Executes GenerateInvoiceUC   │
└────────────────────────────────┘         └────────────────────────────────┘
```

#### Step 1: Feature A publishes an immutable Event Record
```java
package com.example.app.ordering.domain.event;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;

public record OrderPlacedEvent(UUID orderId, UUID customerId, BigDecimal total, Instant occurredAt) {}
```

#### Step 2: Feature B handles the event in its Infrastructure layer
```java
package com.example.app.billing.infrastructure.event;

import com.example.app.billing.application.api.GenerateInvoiceUseCase;
import com.example.app.billing.application.api.GenerateInvoiceUseCase.GenerateInvoiceCommand;
import com.example.app.ordering.domain.event.OrderPlacedEvent;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

@Component
public class OrderPlacedEventListener {

    private final GenerateInvoiceUseCase generateInvoiceUseCase;

    public OrderPlacedEventListener(GenerateInvoiceUseCase generateInvoiceUseCase) {
        this.generateInvoiceUseCase = generateInvoiceUseCase;
    }

    @Async
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        var command = new GenerateInvoiceCommand(event.orderId(), event.customerId(), event.total());
        generateInvoiceUseCase.execute(command);
    }
}
```

---

## 4. Encapsulation & Visibility Rules

To prevent accidental cross-feature leakage in Java:

1. **Inbound Ports & DTOs are `public`**: Only interfaces in `application.api` and their command/query `record`s should be public.
2. **Use Case Implementations are Package-Private**: Use case implementations in `application.usecase.impl` should not be public if they reside in the same package, or they must not be directly instantiated outside their module.
3. **Entities are Package-Scoped**: Domain entities (`Order`, `Invoice`) should never be visible to neighboring features.
4. **Shared Kernel is Minimal**: Only place universal, immutable Value Objects in `shared` (e.g., `TenantId`, `CurrencyCode`). Never share mutable entities, database tables, or business services.
