# Framework Isolation: Keeping Domain & Application 100% Pure Java

A core requirement of Clean Architecture is that business logic and use case orchestration must survive framework obsolescence, library upgrades, or technology shifts. 

To achieve this, **neither the `domain` nor the `application` layer is permitted to import or reference any framework code** (such as Spring, Jakarta Persistence, Hibernate, Jackson, or Quarkus).

---

## 1. Architectural Boundary Map

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        INFRASTRUCTURE LAYER                            │
│  • Spring Boot Starters, Web Controllers, REST Endpoints               │
│  • Spring Data Repositories, JPA Entities, Mongo Documents             │
│  • Kafka / RabbitMQ Consumers & Producers                              │
│  • Spring @Configuration Bean Factory Declarations                     │
│  • Object Mappers (Jackson, MapStruct)                                 │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ (implements & configures)
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   APPLICATION LAYER (100% PURE JAVA)                   │
│  • API Interfaces (Inbound Ports)                                      │
│  • Gateway Interfaces (Outbound Ports)                                 │
│  • Core Use Cases & Implementations                                    │
│  • Command & Query Transfer Records                                    │
│  🚫 NO @Service, NO @Component, NO @Transactional, NO @Autowired       │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ (depends on)
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                      DOMAIN LAYER (100% PURE JAVA)                     │
│  • Entities & Aggregates                                               │
│  • Value Objects (immutable records with compact constructors)         │
│  • Domain Events & Algebraic Result Types (sealed interfaces)          │
│  • Domain Exceptions                                                   │
│  🚫 NO Framework Annotations or External Dependencies                  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Why the Application Layer Must Be Framework-Free

Many architectures correctly isolate the `domain` layer but mistakenly allow Spring annotations (`@Service`, `@Transactional`, `@Autowired`) inside `application` use cases. 

This violates Clean Architecture principles because:
1. **Coupling to Container Semantics**: Use cases become dependent on Spring's proxy generation and lifecycle management.
2. **Hidden Execution Dependencies**: Magic annotations like `@Transactional` obscure where database boundaries open and close.
3. **Impeded Testability**: Use case implementations cannot be tested with pure, instantaneous Java unit tests without loading mock container contexts.

---

## 3. Pure Java Use Case Implementation (Application Layer)

A use case implementation is a plain Java class implementing an Inbound Port and taking Outbound Ports via standard constructor parameters:

```java
package br.com.example.application.usecase.impl;

import br.com.example.application.api.RegisterCustomerApi;
import br.com.example.domain.repository.CustomerRepository;
import br.com.example.application.gateway.EventPublisherGateway;
import br.com.example.domain.model.Customer;
import br.com.example.domain.model.Email;
import br.com.example.domain.model.Result;
import br.com.example.domain.model.CustomerRegistrationError;
import java.util.Objects;
import java.util.UUID;

// PURE JAVA CLASS: No @Service, No @Component, No @Transactional
public final class RegisterCustomerUseCaseImpl implements RegisterCustomerApi {

    private final CustomerRepository customerRepository;
    private final EventPublisherGateway eventPublisher;

    public RegisterCustomerUseCaseImpl(
            CustomerRepository customerRepository, 
            EventPublisherGateway eventPublisher) {
        this.customerRepository = Objects.requireNonNull(customerRepository, "customerRepository is required");
        this.eventPublisher = Objects.requireNonNull(eventPublisher, "eventPublisher is required");
    }

    @Override
    public Result<CustomerRegisteredResponse, CustomerRegistrationError> execute(RegisterCustomerCommand command) {
        Email email = new Email(command.email());

        if (customerRepository.existsByEmail(email)) {
            return Result.failure(new CustomerRegistrationError.EmailAlreadyInUse(email.value()));
        }

        Customer customer = new Customer(UUID.randomUUID(), command.name(), email);
        customerRepository.save(customer);
        eventPublisher.publishCustomerRegistered(customer.getId(), customer.getEmail());

        return Result.success(new CustomerRegisteredResponse(customer.getId(), customer.getName(), customer.getEmail().value()));
    }
}
```

---

## 4. Quarantining Spring to Infrastructure

All Spring bean declarations and framework configurations reside exclusively in `infrastructure.config`:

```java
package br.com.example.infrastructure.config;

import br.com.example.application.api.RegisterCustomerApi;
import br.com.example.domain.repository.CustomerRepository;
import br.com.example.application.gateway.EventPublisherGateway;
import br.com.example.application.usecase.impl.RegisterCustomerUseCaseImpl;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration(proxyBeanMethods = false)
public class ApplicationBeansConfig {

    @Bean
    public RegisterCustomerApi registerCustomerApi(
            CustomerRepository customerRepository,
            EventPublisherGateway eventPublisher) {
        // Pure Java interactor instantiated cleanly inside Spring's IoC container
        return new RegisterCustomerUseCaseImpl(customerRepository, eventPublisher);
    }
}
```

---

## 5. Decoupled Persistence & Entity Mapping

The database schema (JPA `@Entity` or MongoDB `@Document`) must **never** leak into the application or domain layers.

### Infrastructure JPA Entity:
```java
package br.com.example.infrastructure.database.entity;

import jakarta.persistence.*;
import java.util.UUID;

@Entity
@Table(name = "customers")
public class CustomerJpaEntity {
    @Id
    private UUID id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(nullable = false, unique = true)
    private String email;

    // Default constructor, getters, and setters required strictly by JPA
}
```

### Infrastructure Port Adapter:
```java
package br.com.example.infrastructure.database.adapter;

import br.com.example.domain.repository.CustomerRepository;
import br.com.example.domain.model.Customer;
import br.com.example.domain.model.Email;
import br.com.example.infrastructure.database.entity.CustomerJpaEntity;
import br.com.example.infrastructure.database.repository.SpringDataCustomerRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public class CustomerRepositoryAdapter implements CustomerRepository {

    private final SpringDataCustomerRepository springDataRepo;

    public CustomerRepositoryAdapter(SpringDataCustomerRepository springDataRepo) {
        this.springDataRepo = springDataRepo;
    }

    @Override
    public boolean existsByEmail(Email email) {
        return springDataRepo.existsByEmail(email.value());
    }

    @Override
    public void save(Customer customer) {
        CustomerJpaEntity entity = CustomerMapper.toEntity(customer);
        springDataRepo.save(entity);
    }

    @Override
    public Optional<Customer> findById(UUID id) {
        return springDataRepo.findById(id).map(CustomerMapper::toDomain);
    }
}
```

By maintaining this strict boundary, upgrading Spring Boot, swapping from SQL to MongoDB, or moving to a serverless runtime requires editing **only** the `infrastructure` package. The core business rules and use cases remain 100% untouched.
