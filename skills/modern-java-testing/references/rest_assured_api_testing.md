# API Testing with RestAssured in Modern Java

RestAssured provides a domain-specific language (DSL) for testing HTTP REST endpoints in Spring Boot integration tests. Combined with modern Java `record`s and dynamic test ports, it allows concise, expressive, and robust end-to-end verification.

---

## 1. The Fluent `given() -> when() -> then()` Pattern

RestAssured follows standard BDD structure:
- **`given()`**: Request preconditions (headers, query parameters, auth tokens, body payloads).
- **`when()`**: The HTTP action executed (`post()`, `get()`, `put()`, `delete()`).
- **`then()`**: Response assertions (status codes, headers, JSON body assertions).

```java
@Test
@DisplayName("Should successfully create order and return 201 Created")
void shouldCreateOrderSuccessfully() {
    var request = new CreateOrderRequest(UUID.randomUUID(), List.of("ITEM-1", "ITEM-2"), "USD");

    RestAssured
        .given()
            .contentType(ContentType.JSON)
            .body(request)
        .when()
            .post("/api/v1/orders")
        .then()
            .statusCode(HttpStatus.CREATED.value())
            .contentType(ContentType.JSON)
            .body("status", equalTo("PENDING"))
            .body("items", hasSize(2))
            .body("orderId", notNullValue());
}
```

---

## 2. Deserializing Directly into Java Records

Never use untyped `Map<String, Object>` or verbose manual string parsing. Extract JSON response bodies directly into strongly typed, immutable Java `record`s:

```java
@Test
@DisplayName("Should fetch customer details as immutable record")
void shouldFetchCustomerDetails() {
    CustomerResponse customer = RestAssured
        .given()
            .header("Authorization", "Bearer " + validAuthToken)
        .when()
            .get("/api/v1/customers/{id}", customerId)
        .then()
            .statusCode(HttpStatus.OK.value())
            .extract()
            .as(CustomerResponse.class); // Deserializes directly into Java record

    assertThat(customer.id()).isEqualTo(customerId);
    assertThat(customer.email()).isEqualTo("alice@example.com");
    assertThat(customer.status()).isEqualTo("ACTIVE");
}
```

---

## 3. Reusable Specifications (`RequestSpecification` & `ResponseSpecification`)

Eliminate boilerplate across test classes by composing reusable request and response specifications:

```java
public abstract class AbstractRestAssuredTest extends AbstractIntegrationTest {

    protected RequestSpecification authenticatedJsonSpec;
    protected ResponseSpecification standardErrorSpec;

    @BeforeEach
    void setUpSpecifications() {
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = serverPort;
        RestAssured.enableLoggingOfRequestAndResponseIfValidationFails();

        // Reusable authenticated JSON request spec
        authenticatedJsonSpec = new RequestSpecBuilder()
            .setContentType(ContentType.JSON)
            .setAccept(ContentType.JSON)
            .addHeader("Authorization", "Bearer " + getValidToken())
            .addHeader("X-Correlation-ID", UUID.randomUUID().toString())
            .build();

        // Reusable error response spec
        standardErrorSpec = new ResponseSpecBuilder()
            .expectContentType(ContentType.JSON)
            .expectHeader("X-Content-Type-Options", "nosniff")
            .build();
    }
}
```

### Usage with Specifications:
```java
@Test
void shouldRejectUnauthorizedRequest() {
    RestAssured
        .given()
            .spec(authenticatedJsonSpec)
            .body(new InvalidPayload())
        .when()
            .post("/api/v1/secure-endpoint")
        .then()
            .spec(standardErrorSpec)
            .statusCode(HttpStatus.BAD_REQUEST.value())
            .body("errorCode", equalTo("INVALID_PAYLOAD"));
}
```

---

## 4. Best Practices Checklist for RestAssured

1. **Always Bind to Dynamic Ports**: Never hardcode port 8080. Inject `@LocalServerPort` and assign `RestAssured.port = serverPort` in `@BeforeEach`.
2. **Enable Logging on Failure**: Call `RestAssured.enableLoggingOfRequestAndResponseIfValidationFails()` in base setup. This prints full request headers, payload, and response bodies only when a test fails, preventing noisy CI logs on successful runs.
3. **Validate HTTP Semantics**: Check HTTP status code, `Content-Type`, and relevant security headers alongside the body.
4. **Use Text Blocks for Inline JSON**: When testing edge-case payloads or raw payloads, use Java Text Blocks (`"""`) instead of cumbersome escaped strings.
5. **Use `.noContentType()` on Bodyless Requests (`GET`, `DELETE`)**: HTTP `GET` and `DELETE` requests should not send a `Content-Type` header. In modern runtimes (Groovy 5+), RestAssured's default delegation may attempt to resolve request content types with null arguments; calling `given().noContentType().when().get(...)` enforces clean HTTP semantics and prevents runtime delegation issues.
