# Dependency Hygiene, Code Style & Idioms

Modern Java has eliminated the need for heavy boilerplate-generation libraries and external utility packages. This guide defines standard style rules and dependency hygiene.

---

## 1. The Anti-Lombok Standard

Java `record`s, compact constructors, pattern matching, and sealed types replace virtually all reasons Lombok was historically adopted.

### Banned Lombok Annotations
- 🚫 `@Getter` / `@Setter`: Violates encapsulation and promotes mutable state. Use `record` accessors or immutable class designs.
- 🚫 `@ToString` / `@EqualsAndHashCode`: Automatically and correctly implemented by `record`.
- 🚫 `@Data` / `@Value`: Completely superseded by `record`.
- 🚫 `@NoArgsConstructor` / `@AllArgsConstructor`: Replaced by native record constructors and compact constructors.
- 🚫 `@Builder` on Records: Obscures record semantics and creates unnecessary synthetic classes. Use static factory methods or native fluent constructors.

### Permitted Exception
- `@Slf4j`: Permitted only if the project does not use Java's native `System.Logger`.

---

## 2. Type Inference with `var`

The `var` keyword improves readability when types are redundant, but degrades maintainability when overused.

### ✅ Allowed: Self-Explanatory Context
```java
var orders = new ArrayList<Order>();
var customer = Customer.of(id);
var path = Path.of("config/app.json");
```

### ❌ Prohibited: Obscure Return Types
```java
// What does this return? Order? Optional<Order>? UUID?
var result = orderService.process(command); 

// Better:
OrderPlacedEvent event = orderService.process(command);
```

---

## 3. Streams API & Collection Idioms

### Modern `Stream.toList()`
Never use `Collectors.toList()` for general stream terminal operations:

```java
// ❌ Legacy and mutable:
List<String> names = users.stream().map(User::name).collect(Collectors.toList());

// ✅ Modern Java (unmodifiable and concise):
List<String> names = users.stream().map(User::name).toList();
```

### Method References & Clean Pipelines
Prefer method references over multi-statement inline lambdas:

```java
// ❌ Cluttered inline lambda:
list.stream()
    .filter(item -> item != null && item.isActive() && item.price().compareTo(BigDecimal.ZERO) > 0)
    .toList();

// ✅ Clean and readable:
list.stream()
    .filter(OrderItem::isEligibleForProcessing)
    .toList();
```

---

## 4. Text Blocks & String Formatting

### Text Blocks (`"""`)
Use text blocks for multiline payloads (SQL queries, JSON, HTML templates) without escape characters:

```java
String jsonSchema = """
    {
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "type": "object",
      "properties": {
        "id": { "type": "string" },
        "amount": { "type": "number" }
      },
      "required": ["id", "amount"]
    }
    """;
```

### Formatting with `String.formatted()`
Use `String.formatted()` instead of legacy `String.format()` or excessive string concatenation:

```java
String message = "User %s attempted unauthorized access at %s".formatted(userId, Instant.now());
```

---

## 5. Native Standard Library Over Third Parties

Rely strictly on the JDK standard library instead of pulling in Apache Commons or Guava:

| Requirement | ❌ Avoid Third-Party / Anti-Pattern | ✅ Use Native JDK |
| :--- | :--- | :--- |
| Null Checks | `Validate.notNull()` | `Objects.requireNonNull(val, "msg")` |
| Bounds Checking | `if (idx < 0 || idx >= size)` | `Objects.checkIndex(idx, size)` |
| File I/O | `FileUtils.readFileToString()` / `IOUtils` | `Files.readString(path)`, `InputStream.transferTo()` |
| Path Manipulation | `FilenameUtils` | `Path.of()`, `Path.resolve()` |
| Date & Time | Joda-Time / `java.util.Date` | `java.time.Instant`, `LocalDate`, `ZonedDateTime` |
| Collection Creation | `Arrays.asList(a, b)` (mutable elements) | `List.of(a, b)` (immutable) |
| Empty / Fixed Collections | Guava `ImmutableList.of()` | `List.of()`, `Set.of()`, `Map.of()` |
| Defensive Copying | `new ArrayList<>(list)` / Guava `ImmutableList.copyOf()` | `List.copyOf(list)`, `Set.copyOf(set)`, `Map.copyOf(map)` |
| String Checks | `StringUtils.isBlank()` | `str.isBlank()`, `str.strip()` |

---

## 6. Explicit Imports Over Inline Qualified Names (FQCN/PQCN)

Avoid writing Fully/Package-Qualified Class Names (FQCN/PQCN) inline within code. Always declare explicit, clean `import` statements at the top of the file.

### ❌ Avoid: Inline Qualified Names
```java
public java.util.concurrent.CompletableFuture<java.util.UUID> process(java.time.Instant timestamp) {
    return java.util.concurrent.CompletableFuture.supplyAsync(() -> java.util.UUID.randomUUID());
}
```

### ✅ Modern Java: Explicit Top-Level Imports
```java
import java.time.Instant;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;

public CompletableFuture<UUID> process(Instant timestamp) {
    return CompletableFuture.supplyAsync(UUID::randomUUID);
}
```

> [!NOTE]
> The only permitted exception for inline FQCN is resolving an unavoidable naming collision between two classes sharing the exact same simple name within the same compilation unit (e.g., `java.util.Date` vs `java.sql.Date`).

