# Anti-Flakiness & Determinism in Java Testing

A "flaky test" is a test that passes or fails non-deterministically without code changes. Flaky tests erode team confidence in CI, mask real bugs, and slow delivery.

---

## 1. Eliminate `Thread.sleep()`: Mandate Awaitility

### 🚫 The `Thread.sleep()` Trap:
- Setting arbitrary sleep durations (`Thread.sleep(2000)`) either wastes build time or fails intermittently when CI runners experience CPU throttling or high I/O latency.

### ✅ The Awaitility Pattern:
Use **Awaitility** to poll for conditions and assertions asynchronously with exponential backoff:

```java
import org.awaitility.Awaitility;
import static java.util.concurrent.TimeUnit.SECONDS;
import static org.assertj.core.api.Assertions.assertThat;

@Test
@DisplayName("Should process asynchronous outbox event within 5 seconds")
void shouldProcessOutboxEvent() {
    triggerAsyncBatchJob();

    // Polls automatically until assertions pass or timeout expires
    Awaitility.await()
        .atMost(5, SECONDS)
        .pollInterval(100, java.util.concurrent.TimeUnit.MILLISECONDS)
        .untilAsserted(() -> {
            Optional<OutboxEvent> event = outboxRepository.findByAggregateId(orderId);
            assertThat(event).isPresent();
            assertThat(event.get().getStatus()).isEqualTo(OutboxStatus.PROCESSED);
        });
}
```

---

## 2. Time Determinism via `java.time.Clock` Injection

Never call `Instant.now()`, `LocalDate.now()`, or `System.currentTimeMillis()` directly in domain entities or use cases. Doing so causes tests to fail around midnight, during month/year transitions, or across different server timezones.

### Production Class (Constructor Injection):
```java
public class TokenRenewalService {
    private final Clock clock; // Injected Clock

    public TokenRenewalService(Clock clock) {
        this.clock = Objects.requireNonNull(clock);
    }

    public boolean isExpired(Instant expirationTime) {
        return Instant.now(clock).isAfter(expirationTime);
    }
}
```

### Deterministic Test (Fixed Clock):
```java
@Test
@DisplayName("Should detect expired token accurately")
void shouldDetectExpiredToken() {
    Instant frozenNow = Instant.parse("2026-09-26T12:00:00Z");
    Clock fixedClock = Clock.fixed(frozenNow, ZoneOffset.UTC);

    TokenRenewalService service = new TokenRenewalService(fixedClock);

    Instant pastToken = frozenNow.minusSeconds(60);
    Instant futureToken = frozenNow.plusSeconds(60);

    assertThat(service.isExpired(pastToken)).isTrue();
    assertThat(service.isExpired(futureToken)).isFalse();
}
```

---

## 3. Resetting Stateful Infrastructure in `@BeforeEach`

Stateful components like resilience circuit breakers, rate limiters, or distributed locks (ShedLock) retain state across tests if not explicitly reset:

```java
public abstract class AbstractIntegrationTest {

    @Autowired(required = false)
    protected CircuitBreakerRegistry circuitBreakerRegistry;

    @Autowired(required = false)
    protected RateLimiterRegistry rateLimiterRegistry;

    @BeforeEach
    void resetStatefulInfrastructure() {
        if (circuitBreakerRegistry != null) {
            circuitBreakerRegistry.getAllCircuitBreakers().forEach(CircuitBreaker::reset);
        }
        if (rateLimiterRegistry != null) {
            rateLimiterRegistry.getAllRateLimiters().forEach(RateLimiter::drainPermissions);
        }
    }
}
```

---

## 4. Random Port Allocation & Dynamic Server URLs

Never hardcode ports (`8080`, `27017`) in tests. Port collisions occur whenever tests run in parallel or on shared CI build agents.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public abstract class BaseWebIntegrationTest {

    @LocalServerPort
    protected int serverPort;

    @BeforeEach
    void configureRestAssured() {
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = serverPort;
        RestAssured.enableLoggingOfRequestAndResponseIfValidationFails();
    }
}
```
