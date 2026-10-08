# Use Cases, Ports and Framework-Free Application Layer

The application layer coordinates the execution of domain operations to fulfill specific business use cases. In Clean Architecture, **the application layer is 100% pure Java and must never import or depend on frameworks** (no Spring `@Service`, `@Component`, `@Transactional`, or Jakarta/JPA annotations).

---

## 1. Defining Clean Ports

Ports are interfaces that define input and output boundaries:

### Inbound Port (API Interface)
Inbound ports define the contract for invoking a use case. Data is passed via immutable `record` commands and queries:

```java
package br.com.example.application.api;

import br.com.example.domain.model.Result;
import br.com.example.domain.model.OrderError;
import java.math.BigDecimal;
import java.util.List;
import java.util.UUID;

public interface PlaceOrderApi {
    record OrderItemDto(String productId, int quantity, BigDecimal unitPrice) {}
    record PlaceOrderCommand(UUID customerId, List<OrderItemDto> items, String currency) {}
    record OrderCreatedResponse(UUID orderId, BigDecimal total) {}

    Result<OrderCreatedResponse, OrderError> execute(PlaceOrderCommand command);
}
```

### Outbound Port (Gateway / Repository Interface)
Outbound ports define contracts for secondary interactions (databases, payment processors, external microservices). These interfaces are owned by the domain/application layer, not by infrastructure:

```java
package br.com.example.domain.repository;

import br.com.example.domain.model.Order;
import java.util.Optional;
import java.util.UUID;

public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(UUID id);
}
```

```java
package br.com.example.application.gateway;

import br.com.example.domain.model.Money;
import br.com.example.domain.model.PaymentReceipt;
import br.com.example.domain.model.Result;

public interface PaymentGateway {
    sealed interface PaymentGatewayError {
        record NetworkTimeout(String gateway) implements PaymentGatewayError {}
        record CardDeclined(String reason) implements PaymentGatewayError {}
    }

    Result<PaymentReceipt, PaymentGatewayError> charge(Money amount, String paymentMethodToken);
}
```

---

## 2. Implementing the Use Case

The use case implementation is a plain Java class that implements the inbound port and depends only on outbound port interfaces via constructor injection:

```java
package br.com.example.application.usecase.impl;

import br.com.example.application.api.PlaceOrderApi;
import br.com.example.domain.repository.OrderRepository;
import br.com.example.application.gateway.PaymentGateway;
import br.com.example.domain.model.Order;
import br.com.example.domain.model.OrderError;
import br.com.example.domain.model.Result;
import java.util.Objects;

// PURE JAVA: Strictly NO @Service, NO @Component, NO @Transactional
public final class PlaceOrderUseCaseImpl implements PlaceOrderApi {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;

    // Pure constructor injection on final fields
    public PlaceOrderUseCaseImpl(OrderRepository orderRepository, PaymentGateway paymentGateway) {
        this.orderRepository = Objects.requireNonNull(orderRepository, "orderRepository must not be null");
        this.paymentGateway = Objects.requireNonNull(paymentGateway, "paymentGateway must not be null");
    }

    @Override
    public Result<OrderCreatedResponse, OrderError> execute(PlaceOrderCommand command) {
        Order order = Order.create(command.customerId());
        orderRepository.save(order);
        return Result.success(new OrderCreatedResponse(order.getId(), order.getTotal()));
    }
}
```

---

## 3. Decoupling the Framework (Spring IoC Wiring in Infrastructure)

Because interactors have zero Spring annotations, Spring instantiates them via dedicated `@Configuration` classes in `infrastructure.config`:

```java
package br.com.example.infrastructure.config;

import br.com.example.application.api.PlaceOrderApi;
import br.com.example.domain.repository.OrderRepository;
import br.com.example.application.gateway.PaymentGateway;
import br.com.example.application.usecase.impl.PlaceOrderUseCaseImpl;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration(proxyBeanMethods = false)
public class UseCaseBeanConfiguration {

    @Bean
    public PlaceOrderApi placeOrderApi(
            OrderRepository orderRepository,
            PaymentGateway paymentGateway) {
        return new PlaceOrderUseCaseImpl(orderRepository, paymentGateway);
    }
}
```

---

## 4. Transaction Boundaries Without Framework Coupling

Rather than putting `@Transactional` on the use case interactor, manage transaction boundaries in infrastructure:
1. **At the Adapter / Port Level**: The repository adapter handles persistence within an infrastructure transaction.
2. **Decorator / Proxy in Infrastructure**: A Spring `@Configuration` bean can wrap the interactor with a transactional decorator, keeping the core class 100% agnostic.
