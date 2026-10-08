# Meaningful Coverage: Pitest, Instancio & ArchUnit

High line coverage is often a vanity metric: code can be executed 100% without verifying actual business invariants or edge cases. A professional testing strategy focuses on **test effectiveness**, **mutation survival**, and **architectural fitness**.

---

## 1. Mutation Testing with Pitest (PIT)

### Why Line Coverage is Insufficient:
A test that invokes a method but asserts nothing achieves 100% line coverage in JaCoCo, yet provides zero bug detection.

### How Mutation Testing Works:
1. Pitest modifies compiled bytecode (e.g. changing `amount > 0` to `amount >= 0`, negating conditionals, or returning `null`).
2. It runs the relevant unit tests against the mutated bytecode.
3. If at least one test **fails**, the mutant is **killed** (Success ✅).
4. If all tests pass, the mutant **survives** (Defect in test suite ❌).

### Running Pitest in Gradle:
```groovy
// build.gradle
plugins {
    id 'info.solidsoft.pitest' version '1.15.0'
}

pitest {
    targetClasses = ['com.example.app.**.domain.*', 'com.example.app.**.application.*']
    pitestVersion = '1.16.0'
    junit5PluginVersion = '1.2.1'
    threads = Runtime.runtime.availableProcessors()
    outputFormats = ['XML', 'HTML']
    mutationThreshold = 80 // Fails build if mutation score is under 80%
}
```

---

## 2. Randomized Test Data with Instancio

Writing manual test fixtures with dozens of constructor arguments leads to test bloat and hidden bias. **Instancio** generates deep, realistic object graphs populated with non-null test data:

```java
import org.instancio.Instancio;
import static org.instancio.Select.field;

@Test
@DisplayName("Should calculate total order amount across generated order lines")
void shouldCalculateTotalAcrossOrderLines() {
    // Generates a fully populated, immutable Order record with 5 random items
    Order order = Instancio.of(Order.class)
        .generate(field(Order::items), gen -> gen.collection().size(5))
        .set(field(Order::status), OrderStatus.PENDING)
        .create();

    BigDecimal total = order.calculateTotal();

    assertThat(total).isGreaterThan(BigDecimal.ZERO);
}
```

---

## 3. Exhaustive Boundary Testing with `@ParameterizedTest`

Combine `@ParameterizedTest` with `@ValueSource`, `@CsvSource`, and `@MethodSource` to test boundary values without duplicating test methods:

```java
@ParameterizedTest(name = "Value ''{0}'' should be rejected as invalid email")
@ValueSource(strings = {"", "   ", "plainaddress", "missing@domain", "@missinguser.com"})
@NullAndEmptySource
void shouldRejectInvalidEmails(String invalidEmail) {
    assertThatThrownBy(() -> new Email(invalidEmail))
        .isInstanceOf(IllegalArgumentException.class);
}

@ParameterizedTest
@CsvSource({
    "100.00, 10, 90.00",
    "50.00, 20, 40.00",
    "200.00, 0, 200.00"
})
void shouldApplyPercentageDiscountCorrectly(BigDecimal base, int percentage, BigDecimal expected) {
    var discount = new DiscountPolicy.PercentageOff(BigDecimal.valueOf(percentage));
    var result = pricingService.applyDiscount(base, discount);

    assertThat(result).isEqualByComparingTo(expected);
}
```

---

## 4. Architectural Fitness Testing with ArchUnit

ArchUnit verifies that the architectural rules defined in Clean and Screaming Architecture are strictly upheld as automated test assertions:

### Rule 1: Domain & Application Pure Java (Zero Framework Leakage)
```java
import com.tngtech.archunit.core.importer.ImportOption;
import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

@AnalyzeClasses(packages = "com.example.app", importOptions = ImportOption.DoNotIncludeTests.class)
class ArchitectureFitnessTest {

    @ArchTest
    static final ArchRule domainMustNotDependOnFrameworks =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "org.springframework..",
                "jakarta.persistence..",
                "com.fasterxml.jackson.."
            );

    @ArchTest
    static final ArchRule applicationMustNotDependOnFrameworks =
        noClasses().that().resideInAPackage("..application..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "org.springframework..",
                "jakarta.persistence..",
                "com.fasterxml.jackson.."
            );

    @ArchTest
    static final ArchRule featuresMustNotDirectlyImportNeighboringEntities =
        noClasses().that().resideInAPackage("com.example.app.ordering..")
            .should().dependOnClassesThat().resideInAPackage("com.example.app.billing.domain..");
}
```
If a developer accidentally adds `@Service` to a use case or imports a neighboring domain entity, the test immediately fails the build.
