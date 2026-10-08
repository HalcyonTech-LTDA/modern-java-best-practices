# Foreign Function & Memory (FFM) API, Buffers & Modular Encapsulation

Modern Java provides safe, performant alternatives to legacy off-heap memory management and native interoperability, as well as strict modular encapsulation.

---

## 1. Foreign Function & Memory (FFM) API

The FFM API (`java.lang.foreign.*`) provides pure Java interoperability with foreign code (C/C++) and native off-heap memory, completely replacing legacy JNI and dangerous `sun.misc.Unsafe`.

### Off-Heap Memory Allocation with `Arena`
`Arena` manages the lifecycle of off-heap memory segments deterministically:

```java
import java.lang.foreign.Arena;
import java.lang.foreign.MemorySegment;
import java.lang.foreign.ValueLayout;

public void processOffHeapData() {
    // Arena is auto-closed via try-with-resources, preventing memory leaks
    try (Arena arena = Arena.ofConfined()) {
        // Allocate off-heap memory for 1000 integers
        MemorySegment segment = arena.allocate(ValueLayout.JAVA_INT, 1000);

        for (int i = 0; i < 1000; i++) {
            segment.setAtIndex(ValueLayout.JAVA_INT, i, i * 2);
        }

        int sample = segment.getAtIndex(ValueLayout.JAVA_INT, 50);
        // segment memory is freed immediately upon exiting the block
    }
}
```

### Calling Native C Functions (Replacing JNI)
```java
import java.lang.foreign.*;
import java.lang.invoke.MethodHandle;

public long callNativeStrlen(String input) throws Throwable {
    Linker linker = Linker.nativeLinker();
    SymbolLookup stdlib = linker.defaultLookup();

    MemorySegment strlenAddress = stdlib.find("strlen").orElseThrow();
    FunctionDescriptor descriptor = FunctionDescriptor.of(ValueLayout.JAVA_LONG, ValueLayout.ADDRESS);
    MethodHandle strlen = linker.downcallHandle(strlenAddress, descriptor);

    try (Arena arena = Arena.ofConfined()) {
        MemorySegment cString = arena.allocateFrom(input);
        return (long) strlen.invokeExact(cString);
    }
}
```

---

## 2. High-Performance Direct Buffers & Channels

For streaming high-volume network payloads or file transfers:

- Use **Direct ByteBuffers** (`ByteBuffer.allocateDirect(capacity)`) to interact with kernel page caches without intermediate JVM heap copying.
- Use **`java.nio.channels.FileChannel`** and **`SocketChannel`** for non-blocking and transfer-to / transfer-from zero-copy operations:
  ```java
  try (FileChannel sourceChannel = FileChannel.open(sourcePath, StandardOpenOption.READ);
       FileChannel destChannel = FileChannel.open(destPath, StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {
      // Direct kernel-level transfer without copying into user-space heap
      sourceChannel.transferTo(0, sourceChannel.size(), destChannel);
  }
  ```

---

## 3. Modular Encapsulation (JPMS) & Safe Reflection

### Minimal Public API Exports
In projects utilizing Java Modules (`module-info.java`), export **only** the API contracts intended for consumption:

```java
module br.com.example.billing {
    // Export only public inbound ports and data transfer records
    exports br.com.example.application.api;
    exports br.com.example.application.dto;

    // Do NOT export internal interactor or infrastructure packages
    // opens is avoided unless required for a specific serializer/framework
}
```

### Prohibiting Encapsulation-Breaking Reflection
- Never invoke `setAccessible(true)` to access private fields or bypass package boundaries.
- If dynamic method dispatch or property lookup is necessary, utilize the modern, strongly typed `java.lang.invoke.MethodHandles` API:
  ```java
  MethodHandles.Lookup lookup = MethodHandles.lookup();
  MethodHandle handle = lookup.findVirtual(Order.class, "total", MethodType.methodType(BigDecimal.class));
  BigDecimal total = (BigDecimal) handle.invokeExact(order);
  ```
