# Eliminating Carrier Thread Pinning with `ReentrantLock`

Virtual threads are user-mode threads scheduled by the JVM on a pool of underlying operating system platform threads (known as "carrier threads"). High-throughput concurrency relies on virtual threads being able to unmount from their carrier threads during blocking I/O operations.

---

## 1. What is Pinning?

When a virtual thread cannot unmount during a blocking operation, it remains **pinned** to its carrier thread. If many virtual threads become pinned simultaneously, the carrier pool becomes exhausted, reducing system throughput down to traditional thread pool limits.

### Primary Causes of Pinning:
1. Executing a blocking operation (network call, database query, sleep) inside a **`synchronized` block or method**.
2. Calling blocking operations inside native JNI or foreign function call frames.

> **Note**: Pinning inside `synchronized` only hurts throughput if the operation blocks on I/O. In-memory, non-blocking synchronized sections that execute in nanoseconds do not harm system performance.

---

## 2. Diagnosing Pinning via JVM Flags

To detect when carrier threads are pinned in development or staging:

```bash
# Print a full stack trace whenever a virtual thread is pinned while blocking
java -Djdk.tracePinnedThreads=full -jar app.jar

# Or print a single-line summary stack trace:
java -Djdk.tracePinnedThreads=short -jar app.jar
```

---

## 3. Migration: Replacing `synchronized` with `ReentrantLock`

`java.util.concurrent.locks.ReentrantLock` fully supports virtual thread unmounting during blocking calls.

### ❌ Anti-Pattern: Blocking I/O inside `synchronized` (Causes Pinning)
```java
public class RemoteConfigManager {
    private final Map<String, String> cache = new HashMap<>();

    public synchronized String getOrLoadConfig(String key) {
        if (!cache.containsKey(key)) {
            // BLOCKING HTTP CALL inside synchronized block -> Carrier Thread Pinning!
            String remoteValue = httpClient.fetchConfig(key);
            cache.put(key, remoteValue);
        }
        return cache.get(key);
    }
}
```

### ✅ Modern Java Pattern: `ReentrantLock` (Allows Clean Unmounting)
```java
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;
import java.util.concurrent.locks.ReentrantLock;

public class RemoteConfigManager {
    private final Map<String, String> cache = new HashMap<>();
    private final ReentrantLock lock = new ReentrantLock();

    public String getOrLoadConfig(String key) {
        lock.lock();
        try {
            if (!cache.containsKey(key)) {
                // Virtual thread unmounts cleanly from carrier thread during blocking call
                String remoteValue = httpClient.fetchConfig(key);
                cache.put(key, remoteValue);
            }
            return cache.get(key);
        } finally {
            lock.unlock();
        }
    }
}
```

---

## 4. Alternative: Non-Blocking Concurrent Collections

Whenever possible, avoid explicit locks entirely by using modern lock-free or concurrent data structures:

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

public class LockFreeConfigManager {
    private final ConcurrentMap<String, String> cache = new ConcurrentHashMap<>();

    public String getOrLoadConfig(String key) {
        // computeIfAbsent is fast, highly concurrent, and avoids manual lock management
        return cache.computeIfAbsent(key, httpClient::fetchConfig);
    }
}
```
