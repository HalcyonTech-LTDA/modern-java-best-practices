# Stream Gatherers (`Stream.gather`)

Stream Gatherers provide custom, reusable intermediate operations in Java Stream pipelines. They cleanly replace stateful loops, manual chunking, and complex `flatMap` constructs.

---

## 1. Built-in Gatherers (`java.util.stream.Gatherers`)

The `Gatherers` utility class provides standard intermediate operations:

### Fixed-Size Windowing (`windowFixed`)
Groups elements into fixed-size batches (e.g., for bulk database inserts or batch HTTP requests):

```java
import java.util.List;
import java.util.stream.Gatherers;

List<String> items = List.of("A", "B", "C", "D", "E");

List<List<String>> batches = items.stream()
    .gather(Gatherers.windowFixed(2))
    .toList();
// Result: [["A", "B"], ["C", "D"], ["E"]]
```

### Sliding Window (`windowSliding`)
Computes a rolling or sliding window over elements (e.g., rolling averages or temporal analysis):

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5);

List<List<Integer>> windows = numbers.stream()
    .gather(Gatherers.windowSliding(3))
    .toList();
// Result: [[1, 2, 3], [2, 3, 4], [3, 4, 5]]
```

### Prefix Scan (`scan`)
Computes running / progressive accumulations without terminating the stream (unlike `reduce`):

```java
List<Integer> values = List.of(10, 20, 30);

List<Integer> runningTotals = values.stream()
    .gather(Gatherers.scan(() -> 0, Integer::sum))
    .toList();
// Result: [10, 30, 60]
```

### Incremental Fold (`fold`)
Accumulates elements into a result container in a stateful, intermediate manner:

```java
List<String> words = List.of("Clean", "Architecture", "Modern", "Java");

List<String> accumulated = words.stream()
    .gather(Gatherers.fold(() -> "", (curr, next) -> curr.isEmpty() ? next : curr + " -> " + next))
    .toList();
```

---

## 2. Implementing a Custom Gatherer

A `Gatherer<T, A, R>` is defined by four cooperative functions:
1. **`initializer()`**: Creates the mutable state object.
2. **`integrator()`**: Consumes input elements, updates state, and optionally pushes downstream items via `Downstream<? super R>`.
3. **`combiner()`**: Merges states during parallel stream evaluation.
4. **`finisher()`**: Emits any remaining state when the stream ends.

### Example: Deduplicate Adjacent Elements
```java
import java.util.concurrent.atomic.AtomicReference;
import java.util.stream.Gatherer;

public final class CustomGatherers {

    public static <T> Gatherer<T, ?, T> deduplicateAdjacent() {
        return Gatherer.ofSequential(
            () -> new AtomicReference<T>(null),
            (state, element, downstream) -> {
                T prev = state.get();
                if (prev == null || !prev.equals(element)) {
                    state.set(element);
                    return downstream.push(element);
                }
                return true; // continue without pushing duplicate
            }
        );
    }
}
```

Usage:
```java
List<String> letters = List.of("A", "A", "B", "B", "B", "A", "C");
List<String> deduped = letters.stream()
    .gather(CustomGatherers.deduplicateAdjacent())
    .toList();
// Result: ["A", "B", "A", "C"]
```
