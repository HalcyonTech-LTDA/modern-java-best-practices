# Performance, Configuration Cache & Java Toolchains

High-performance Gradle builds require deterministic JDK environments (**Java Toolchains**), instant build initialization (**Configuration Cache**), and strict adherence to **Task Configuration Avoidance**.

---

## 1. Java Toolchains

Java Toolchains decouple the JDK used to compile and test your application from the JDK used to execute the Gradle daemon.

### Configuration

```kotlin
// In build-logic/src/main/kotlin/myproject.java-common-conventions.gradle.kts
java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(21))
        vendor.set(JvmVendorSpec.ADOPTIUM) // Optional: enforce specific vendor
    }
}
```

### Benefits
1. **Reproducible Builds Across Environments**: Developers on macOS, Linux, and Windows, as well as CI runners, always compile against the exact same JDK version.
2. **Auto-Provisioning**: If the requested JDK is missing, Gradle can automatically download and configure it from configured toolchain repositories.
3. **Daemon Decoupling**: You can run the Gradle daemon on a modern JVM without forcing the application code to target that exact same runtime.

---

## 2. Production `gradle.properties` Flags

Place these flags in your root `gradle.properties` to maximize build throughput:

```properties
# Enable configuration caching for near-instant build invocations
org.gradle.configuration-cache=true

# Enable build artifact caching across runs and branches
org.gradle.caching=true

# Build independent projects in parallel
org.gradle.parallel=true

# Watch the file system to avoid scanning on every build
org.gradle.vfs.watch=true

# Daemon memory allocation and garbage collection tuning
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
```

---

## 3. Configuration Cache Rules & Anti-Patterns

The Configuration Cache serializes the resolved task execution graph after the configuration phase, skipping configuration entirely on subsequent runs.

### 🚫 The 4 Banned Anti-Patterns

#### Anti-Pattern 1: Eager Task Realization
```kotlin
// 🚫 BANNED: Forces immediate configuration of the task even if not in the execution graph
tasks.create("generateDocs") { ... }
tasks.getByName("test") { ... }

// ✅ MANDATED: Lazy configuration via Task Configuration Avoidance
tasks.register("generateDocs") { ... }
tasks.named("test") { ... }
```

#### Anti-Pattern 2: Accessing `Project` at Execution Time
```kotlin
tasks.register("printVersion") {
    doLast {
        // 🚫 BANNED: 'project' reference cannot be serialized into configuration cache
        println(project.version)
    }
}

// ✅ MANDATED: Capture inputs lazily via Provider/Property APIs
val projectVersion = project.provider { project.version.toString() }
tasks.register("printVersion") {
    inputs.property("version", projectVersion)
    doLast {
        println(projectVersion.get())
    }
}
```

#### Anti-Pattern 3: Undeclared File Inputs or System Environment Reads
Tasks must never read system properties, environment variables, or files inside `doLast` without declaring them as inputs:

```kotlin
abstract class CustomTask : DefaultTask() {
    @get:Input
    abstract val apiKey: Property<String>

    @get:OutputFile
    abstract val destinationFile: RegularFileProperty

    @TaskAction
    fun execute() {
        destinationFile.get().asFile.writeText("Key: ${apiKey.get()}")
    }
}
```

#### Anti-Pattern 4: Mutating Shared Global State
Avoid static mutable variables or sharing mutable references across task boundaries.

---

## 4. Diagnosing & Troubleshooting

When troubleshooting configuration cache issues:

```bash
# Run with configuration cache enabled and inspect errors
./gradlew build --configuration-cache

# If failures occur, open the generated HTML report:
open build/reports/configuration-cache/<report-id>/configuration-cache-report.html
```

The HTML report provides an interactive tree showing the exact task, file, and line number that captured an invalid reference.
