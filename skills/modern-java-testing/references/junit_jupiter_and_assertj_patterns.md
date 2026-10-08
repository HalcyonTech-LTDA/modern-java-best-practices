# JUnit Jupiter & AssertJ Conventions and Patterns

Modern Java automated tests rely on the full capabilities of **JUnit 5 (Jupiter)** and **AssertJ**. This guide outlines standard patterns, idioms, and conventions.

---

## 1. Exclusivity of JUnit 5 (Banning JUnit 4)

Never allow JUnit 4 legacy imports into modern Java projects:

| ❌ Banned JUnit 4 Import | ✅ Modern JUnit Jupiter Equivalent |
| :--- | :--- |
| `org.junit.Test` | `org.junit.jupiter.api.Test` |
| `org.junit.Before` | `org.junit.jupiter.api.BeforeEach` |
| `org.junit.After` | `org.junit.jupiter.api.AfterEach` |
| `org.junit.BeforeClass` | `org.junit.jupiter.api.BeforeAll` |
| `org.junit.AfterClass` | `org.junit.jupiter.api.AfterAll` |
| `org.junit.Ignore` | `org.junit.jupiter.api.Disabled` |
| `org.junit.Assert.*` | `org.assertj.core.api.Assertions.*` |
| `org.junit.runner.RunWith` | `org.junit.jupiter.api.extension.ExtendWith` |

---

## 2. Hierarchical BDD Contexts with `@Nested`

Use `@Nested` inner classes to structure tests logically around system state, operations, or BDD contexts (`Given / When / Then`):

```java
@DisplayName("Customer Account")
class CustomerAccountTest {

    private Customer customer;

    @BeforeEach
    void setUp() {
        customer = new Customer(UUID.randomUUID(), "Alice", new Email("alice@example.com"));
    }

    @Nested
    @DisplayName("When account is active")
    class WhenAccountIsActive {

        @Test
        @DisplayName("Should successfully suspend with a valid reason")
        void shouldSuspendWithValidReason() {
            customer.suspend("Policy violation");
            assertThat(customer.getStatus()).isEqualTo(CustomerStatus.SUSPENDED);
        }

        @Test
        @DisplayName("Should allow updating contact email")
        void shouldAllowEmailUpdate() {
            var newEmail = new Email("new.alice@example.com");
            customer.updateContactEmail(newEmail);
            assertThat(customer.getEmail()).isEqualTo(newEmail);
        }
    }

    @Nested
    @DisplayName("When account is already suspended")
    class WhenAccountIsSuspended {

        @BeforeEach
        void suspendAccount() {
            customer.suspend("Initial suspension");
        }

        @Test
        @DisplayName("Should reject another suspension attempt")
        void shouldRejectSecondSuspension() {
            assertThatThrownBy(() -> customer.suspend("Second attempt"))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("already suspended");
        }

        @Test
        @DisplayName("Should reject email updates while suspended")
        void shouldRejectEmailUpdate() {
            assertThatThrownBy(() -> customer.updateContactEmail(new Email("test@example.com")))
                .isInstanceOf(IllegalStateException.class);
        }
    }
}
```

---

## 3. Human-Readable Test Names (`ReplaceUnderscores`)

Instead of writing verbose `@DisplayName("...")` strings on every single test method, use `@DisplayNameGeneration(DisplayNameGenerator.ReplaceUnderscores.class)` at the class or package level. This automatically converts snake_case method names into readable sentences:

```java
@DisplayNameGeneration(DisplayNameGenerator.ReplaceUnderscores.class)
class OrderCalculationTest {

    @Test
    void should_calculate_total_including_vat_and_discounts() {
        // Test method name is automatically displayed as:
        // "should calculate total including vat and discounts"
    }

    @Test
    void should_reject_negative_quantity_in_order_line() {
        // Displayed as: "should reject negative quantity in order line"
    }
}
```

---

## 4. Safe Filesystem Tests with `@TempDir`

Never write temporary test files into hardcoded paths (like `/tmp` or the current directory). Use JUnit Jupiter's **`@TempDir`** to inject an isolated, automatically created and deleted directory:

```java
class ReportGeneratorTest {

    @Test
    @DisplayName("Should export audit report to CSV file cleanly")
    void shouldExportAuditReport(@TempDir Path tempDir) throws IOException {
        Path reportFile = tempDir.resolve("audit-report.csv");

        reportGenerator.generateReport(testData, reportFile);

        assertThat(Files.exists(reportFile)).isTrue();
        assertThat(Files.readString(reportFile))
            .contains("ORDER_ID,AMOUNT,STATUS")
            .contains("123,450.00,COMPLETED");
        // tempDir and its contents are automatically cleaned up when the test finishes
    }
}
```

---

## 5. Test Classification with `@Tag`

Tag tests to enable selective test execution in CI/CD pipelines (e.g. running fast unit tests on every commit, and slower integration tests on merge):

```java
@Tag("integration")
class OrderDatabaseAdapterIntegrationTest extends AbstractIntegrationTest { ... }

@Tag("architecture")
class ArchitectureFitnessTest { ... }
```

In `build.gradle.kts`:
```kotlin
tasks.named<Test>("test") {
    useJUnitPlatform {
        excludeTags("slow", "integration") // Run only fast unit tests by default
    }
}

tasks.register<Test>("integrationTest") {
    useJUnitPlatform {
        includeTags("integration")
    }
}
```

---

## 6. AssertJ Best Practices

### 1. Fluent Exception Assertions with `assertThatThrownBy`
Avoid the procedural `assertThrows`:
```java
// ❌ Clunky JUnit assertion:
Exception ex = assertThrows(IllegalArgumentException.class, () -> new Money(BigDecimal.valueOf(-10), USD));
assertTrue(ex.getMessage().contains("negative"));

// ✅ Expressive AssertJ assertion:
assertThatThrownBy(() -> new Money(BigDecimal.valueOf(-10), USD))
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessageContaining("negative");
```

### 2. Multi-Field Validation with Soft Assertions (`assertSoftly`)
When asserting multiple fields of an immutable record or DTO, standard assertions fail on the first error, hiding other broken fields. Use `assertSoftly` to inspect all assertions simultaneously:

```java
@Test
void shouldMapCustomerDtoCorrectly() {
    CustomerDto dto = CustomerMapper.toDto(customer);

    SoftAssertions.assertSoftly(softly -> {
        softly.assertThat(dto.id()).isEqualTo(customer.getId().value());
        softly.assertThat(dto.fullName()).isEqualTo("Alice Smith");
        softly.assertThat(dto.email()).isEqualTo("alice@example.com");
        softly.assertThat(dto.status()).isEqualTo("ACTIVE");
    }); // Reports ALL assertion failures together in one clean error message
}
```

---

## 7. Mandatory Test Class Naming Conventions

All automated tests must follow strict naming taxonomy:

| Test Category | Suffix Rule | Target Prefix | Concrete Examples |
| :--- | :--- | :--- | :--- |
| **Unit Tests** | `*Test` | Exact target class name | `CreateCustomerUseCaseTest` tests `CreateCustomerUseCase`<br>`CustomerTest` tests `Customer`<br>`CustomerDatabaseAdapterTest` tests `CustomerDatabaseAdapter` |
| **Integration Tests** | `*IntegrationTest` | Exact component/controller name | `CustomerControllerIntegrationTest` tests `CustomerController`<br>`CustomerSyncJobIntegrationTest` tests `CustomerSyncJob` |
| **Architecture Tests** | `Architecture*Test` | Descriptive fitness scope | `ArchitectureFitnessTest` or `ArchitectureTest` |
