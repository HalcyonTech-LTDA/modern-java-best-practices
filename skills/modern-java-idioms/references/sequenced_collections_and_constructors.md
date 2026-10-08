# Sequenced Collections & Flexible Constructor Bodies

Modern Java provides native, standardized contracts for sequenced data and gives developers total flexibility when constructing objects.

---

## 1. Sequenced Collections (`java.util.SequencedCollection`)

Prior to Sequenced Collections, getting the first or last element of a collection differed across `List`, `Deque`, `LinkedHashSet`, and `SortedSet`. Modern Java standardizes this across all ordered collections.

### Core Interface Hierarchy
- `SequencedCollection<E>` extends `Collection<E>`
- `SequencedSet<E>` extends `Set<E>`, `SequencedCollection<E>`
- `SequencedMap<K, V>` extends `Map<K, V>`

### Essential Methods

| Operation | SequencedCollection | SequencedMap |
| :--- | :--- | :--- |
| First Element | `collection.getFirst()` | `map.firstEntry()` |
| Last Element | `collection.getLast()` | `map.lastEntry()` |
| Add First | `collection.addFirst(e)` | `map.putFirst(k, v)` |
| Add Last | `collection.addLast(e)` | `map.putLast(k, v)` |
| Remove First | `collection.removeFirst()` | `map.pollFirstEntry()` |
| Remove Last | `collection.removeLast()` | `map.pollLastEntry()` |
| Reverse View | `collection.reversed()` | `map.reversed()` |

### Idiomatic Examples
```java
List<String> pipelineStages = new ArrayList<>(List.of("validate", "enrich", "persist"));

// Expressive first and last access
String firstStage = pipelineStages.getFirst();
String lastStage = pipelineStages.getLast();

// Iterating in reverse without manual index math
for (String stage : pipelineStages.reversed()) {
    auditRollback(stage);
}

// Sequenced Map
LinkedHashMap<String, Integer> cache = new LinkedHashMap<>();
cache.putFirst("root", 0);
Map.Entry<String, Integer> oldest = cache.firstEntry();
```

---

## 2. Flexible Constructor Bodies

Flexible constructor bodies allow statements to precede `super(...)` or `this(...)` constructor calls (known as prologue statements), provided they do not access the uninitialized instance (`this`).

### Validation Prior to Super Calls
```java
public class PositiveAccount extends Account {
    public PositiveAccount(BigDecimal initialBalance, AccountId id) {
        // Pre-super validation: Clean and immediate failure
        if (initialBalance == null || initialBalance.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Initial balance must be non-negative");
        }
        super(initialBalance, id);
    }
}
```

### Argument Pre-Computation
```java
public class SecureToken extends Token {
    public SecureToken(byte[] rawSecret) {
        // Compute derived values safely without static helper methods
        byte[] salt = KeyUtils.generateSalt();
        byte[] hashed = KeyUtils.deriveKey(rawSecret, salt);
        super(hashed, salt);
    }
}
```
