# Task Orchestration with Virtual Threads & `CompletableFuture`

> **Production-Strict Notice**: While `StructuredTaskScope` and `ScopedValue` exist in modern Java preview releases, they require `--enable-preview` and remain subject to breaking changes. In production code adhering to stability standards, asynchronous subtask coordination must strictly rely on standardized Java APIs: `CompletableFuture` combined with `Executors.newVirtualThreadPerTaskExecutor()`.

---

## 1. Virtual Thread-Backed `CompletableFuture` Pattern

By default, `CompletableFuture.supplyAsync()` uses the common `ForkJoinPool` (platform threads). When orchestrating I/O-bound tasks (database queries, HTTP clients, external microservices), **always pass an explicit virtual thread executor**:

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public record AggregatedAccountDetails(UserProfile profile, List<Transaction> transactions, CreditScore score) {}

public final class CustomerDashboardService {

    public AggregatedAccountDetails fetchDetails(String customerId) {
        // Disposable virtual thread per task executor
        try (ExecutorService vThreadExecutor = Executors.newVirtualThreadPerTaskExecutor()) {
            
            // Fork child subtasks onto virtual threads
            CompletableFuture<UserProfile> profileFuture = CompletableFuture
                .supplyAsync(() -> userService.getProfile(customerId), vThreadExecutor)
                .orTimeout(2, TimeUnit.SECONDS);

            CompletableFuture<List<Transaction>> txFuture = CompletableFuture
                .supplyAsync(() -> txService.getRecentTransactions(customerId), vThreadExecutor)
                .orTimeout(3, TimeUnit.SECONDS);

            CompletableFuture<CreditScore> scoreFuture = CompletableFuture
                .supplyAsync(() -> creditService.getCreditScore(customerId), vThreadExecutor)
                .orTimeout(2, TimeUnit.SECONDS);

            // Wait for all subtasks to complete
            CompletableFuture.allOf(profileFuture, txFuture, scoreFuture).join();

            // Safe retrieval - all futures are guaranteed done
            return new AggregatedAccountDetails(
                profileFuture.join(),
                txFuture.join(),
                scoreFuture.join()
            );
        } // Executor auto-closes, waiting for or interrupting remaining tasks
    }
}
```

---

## 2. Fail-Fast Coordination (Short-Circuit on First Error)

To achieve fail-fast behavior where the failure of any subtask cancels or bypasses the others:

```java
public <T> CompletableFuture<T> failFastOrAll(List<CompletableFuture<?>> futures, Supplier<T> resultSupplier) {
    CompletableFuture<T> completion = new CompletableFuture<>();

    for (var future : futures) {
        future.whenComplete((_, throwable) -> {
            if (throwable != null) {
                // Complete exceptionally on the first failure
                completion.completeExceptionally(throwable);
                // Cancel remaining sibling tasks
                futures.forEach(f -> f.cancel(true));
            }
        });
    }

    CompletableFuture.allOf(futures.toArray(CompletableFuture[]::new))
        .thenRun(() -> completion.complete(resultSupplier.get()))
        .exceptionally(ex -> {
            completion.completeExceptionally(ex);
            return null;
        });

    return completion;
}
```

---

## 3. First-Responder / Speculative Execution (`anyOf`)

When querying redundant data sources and needing only the fastest valid response:

```java
public WeatherData fetchFastestWeather(Coordinates coords, ExecutorService executor) {
    CompletableFuture<WeatherData> primary = CompletableFuture
        .supplyAsync(() -> primaryWeatherApi.fetch(coords), executor);

    CompletableFuture<WeatherData> secondary = CompletableFuture
        .supplyAsync(() -> secondaryWeatherApi.fetch(coords), executor);

    CompletableFuture<WeatherData> fallback = CompletableFuture
        .supplyAsync(() -> fallbackWeatherApi.fetch(coords), executor);

    // Resolves immediately when the fastest future completes
    Object fastest = CompletableFuture.anyOf(primary, secondary, fallback).join();
    return (WeatherData) fastest;
}
```

---

## 4. Best Practices for Production Concurrency

1. **Scoped Executors via `try-with-resources`**: Always scope your `Executors.newVirtualThreadPerTaskExecutor()` within a try-with-resources block so all submitted tasks are either completed or cleaned up upon exiting the scope.
2. **Explicit Timeouts**: Always attach `.orTimeout(duration, unit)` or `.completeOnTimeout(...)` to prevent rogue network calls from hanging virtual threads indefinitely.
3. **Handle Interruption**: Code executed inside virtual threads should periodically check `Thread.currentThread().isInterrupted()` if performing loops or intensive steps.
