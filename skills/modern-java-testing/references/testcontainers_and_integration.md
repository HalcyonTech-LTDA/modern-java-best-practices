# Integration Testing with Testcontainers & Spring Boot

Integration tests verify that persistence adapters, database queries, migration scripts, and external infrastructure clients function correctly against real runtime dependencies.

---

## 1. The Singleton Container Pattern

Starting a new Docker container per test class (`@Container` instance per class) is the primary cause of slow and resource-exhausted CI pipelines. 

The **Singleton Container Pattern** starts containers once per JVM execution and shares them across all integration test suites:

```java
package com.example.app.infrastructure;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.test.context.ActiveProfiles;
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.utility.DockerImageName;

@ActiveProfiles("test")
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public abstract class AbstractIntegrationTest {

    // Started once per JVM run across all test classes
    @ServiceConnection
    protected static final MongoDBContainer MONGO_CONTAINER = 
        new MongoDBContainer(DockerImageName.parse("mongo:7.0"))
            .withReuse(true);

    static {
        MONGO_CONTAINER.start();
    }

    @LocalServerPort
    protected int serverPort;
}
```

---

## 2. Spring Boot `@ServiceConnection` vs. `@DynamicPropertySource`

In modern Spring Boot (3.1+), **`@ServiceConnection`** replaces verbose `@DynamicPropertySource` blocks. Spring automatically discovers the running container and registers connection properties for supported technologies:
- `MongoDBContainer` -> `spring.data.mongodb.uri`
- `PostgreSQLContainer` -> `spring.datasource.url`, `username`, `password`
- `KafkaContainer` -> `spring.kafka.bootstrap-servers`
- `GenericContainer` (Redis) -> `spring.data.redis.host`, `port`

If custom property binding is required for unsupported containers:
```java
@DynamicPropertySource
static void configureProperties(DynamicPropertyRegistry registry) {
    registry.add("custom.service.url", CUSTOM_CONTAINER::getEndpoint);
}
```

---

## 3. Banishing `@DirtiesContext` & Ensuring Clean State

### 🚫 Why `@DirtiesContext` is an Anti-Pattern:
- Annotating test classes with `@DirtiesContext` instructs Spring to discard and recreate the entire `ApplicationContext` (beans, database pools, thread executors).
- In a suite with 30 integration test classes, this increases test execution time from 20 seconds to 10+ minutes.

### ✅ The Clean State Pattern (Truncate in `@BeforeEach`):
Instead of destroying the Spring context, clean database collections or tables in `@BeforeEach`:

```java
public abstract class AbstractIntegrationTest {

    @Autowired
    protected MongoTemplate mongoTemplate;

    @BeforeEach
    void resetDatabaseState() {
        // Fast, in-place cleanup without restarting context or containers
        for (String collectionName : mongoTemplate.getCollectionNames()) {
            if (!collectionName.startsWith("system.")) {
                mongoTemplate.dropCollection(collectionName);
            }
        }
    }
}
```

For relational databases (PostgreSQL/MySQL), truncate tables or use transactional test rollbacks (`@Transactional` on test methods where auto-commit is not required).

---

## 4. External HTTP Stubs with WireMock

When testing outbound REST client adapters (e.g. payment gateways, credit bureaus), use **WireMock** rather than mocking the HTTP client:

```java
@Test
@DisplayName("Should fetch external bureau report successfully")
void shouldFetchBureauReport() {
    WireMock.stubFor(WireMock.post("/oauth/token")
        .willReturn(WireMock.okJson("""
            { "access_token": "valid-test-token", "expires_in": 3600 }
            """)));

    WireMock.stubFor(WireMock.get("/v1/bureau/12345678900")
        .withHeader("Authorization", WireMock.equalTo("Bearer valid-test-token"))
        .willReturn(WireMock.okJson("""
            { "cpf": "12345678900", "score": 850, "status": "REGULAR" }
            """)));

    BureauReport report = bureauClientAdapter.fetchReport(new Cpf("12345678900"));

    assertThat(report.score()).isEqualTo(850);
}
```
