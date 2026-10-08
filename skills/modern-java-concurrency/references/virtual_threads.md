# Virtual Threads in Modern Java

Virtual threads are lightweight implementations of `java.lang.Thread` managed directly by the Java Virtual Machine rather than the underlying operating system kernel. They are fully standardized and production-ready in modern Java.

---

## 1. Creating and Using Virtual Threads

### Per-Task Executor (Recommended for Servers & Dispatchers)
For batch processors, web request pipelines, and concurrent work dispatchers:

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

// Auto-closes and waits for all submitted tasks to complete
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10_000; i++) {
        final int taskId = i;
        executor.submit(() -> {
            processItem(taskId);
            return null;
        });
    }
}
```

### Direct Virtual Thread Builder
```java
Thread vThread = Thread.ofVirtual()
    .name("order-worker-", 0)
    .start(() -> handleOrder(orderId));
```

---

## 2. Core Rules for Virtual Threads

### 1. Never Pool Virtual Threads
- Traditional platform threads are heavy OS resources (1MB+ memory per thread), requiring bounded pools (`ThreadPoolExecutor`).
- Virtual threads cost only a few hundred bytes initially and are stored in the JVM heap. Pooling them adds memory overhead, contention, and destroys their disposable design.
- **Rule**: Always spawn a new virtual thread per task (`Executors.newVirtualThreadPerTaskExecutor()`).

### 2. Differentiate Workloads
- **I/O-Bound Workloads (Virtual Threads)**:
  - Database queries (JDBC / R2DBC).
  - Outgoing HTTP/REST requests (`HttpClient`).
  - Message brokers (Kafka, RabbitMQ consumers).
  - File reading and writing.
- **CPU-Bound Workloads (Platform Threads)**:
  - Heavy cryptographic hashing, video encoding, complex mathematical simulations.
  - Keep CPU-intensive computation confined to `ForkJoinPool.commonPool()` or a dedicated fixed pool sized to available CPU cores.

---

## 3. Spring Boot Integration

To enable virtual threads universally in Spring Boot:

```properties
# application.properties
spring.threads.virtual.enabled=true
```

When enabled:
- Tomcat, Jetty, and Undertow HTTP request handlers execute on virtual threads.
- `@Async` methods automatically execute on virtual threads.
- Spring TaskExecution and Scheduling use virtual thread executors.
