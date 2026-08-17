# Testing Strategy

## The Testing Pyramid

```
       /\
      /  \        E2E Tests (Few)
     /----\       - Full system
    /      \      - Slow, brittle
   /--------\
  /          \    Integration Tests (Some)
 /------------\   - Multiple components
/              \  - Medium speed
----------------
      Unit Tests (Many)
      - Single unit
      - Fast, isolated
```

## Test Types

### Unit Tests

Test ONE class or function in isolation.

**Characteristics:**
- Fast (milliseconds)
- No external dependencies (mocked)
- Most of your tests should be unit tests

```java
class OrderTest {
    @Test
    void calculatesTotalCorrectly() {
        Order order = new Order();
        order.addItem(new OrderItem(Money.dollars(100)));
        order.addItem(new OrderItem(Money.dollars(50)));

        assertEquals(Money.dollars(150), order.calculateTotal());
    }
}
```

### Integration Tests

Test multiple components together.

**Characteristics:**
- Slower (may use real DB)
- Test boundaries between components
- Fewer than unit tests

```java
class OrderServiceIntegrationTest {
    private Database db;
    private OrderService service;

    @BeforeAll
    void setUp() {
        db = Database.connect();
        service = new OrderService(new PostgresOrderRepo(db));
    }

    @Test
    void savesAndRetrievesOrder() {
        Order order = Order.create(new CustomerId("123"));
        service.save(order);

        Order retrieved = service.findById(order.getId());
        assertEquals(order, retrieved);
    }
}
```

### E2E / Acceptance Tests

Test the entire system from user perspective.

**Characteristics:**
- Slowest
- Most brittle (many moving parts)
- Test critical paths only

```java
class CheckoutFlowTest {
    @Test
    void userCanCompletePurchase() {
        page.goto("/products");
        page.click("[data-testid='add-to-cart']");
        page.click("[data-testid='checkout']");
        page.fill("[name='card']", "4242424242424242");
        page.click("[data-testid='pay']");

        assertEquals("Order Confirmed", page.textContent("h1"));
    }
}
```

---

## Arrange-Act-Assert (AAA)

Structure EVERY test this way:

```java
@Test
void appliesDiscountToPremiumUsers() {
    // ARRANGE - Set up the test world
    User user = new User(AccountType.PREMIUM);
    Cart cart = new Cart(user);
    cart.addItem(new CartItem(Money.dollars(100)));

    // ACT - Execute the behavior under test
    BigDecimal total = cart.calculateTotal();

    // ASSERT - Verify the expected outcome
    assertEquals(new BigDecimal("80"), total); // 20% discount
}
```

### Writing AAA Backwards

Sometimes easier to write in reverse:

1. **Assert first** - What do you want to verify?
2. **Act** - What action produces that result?
3. **Arrange** - What setup is needed?

---

## Test Naming

### Bad: Abstract, Technical

```java
@Test
void shouldWorkCorrectly() { }

@Test
void handlesTheEdgeCase() { }

@Test
void setsTheDataProperty() { }
```

### Good: Concrete Examples, Domain Language

```java
@Test
void calculates20PercentDiscountForPremiumUsers() { }

@Test
void returnsErrorWhenCartIsEmpty() { }

@Test
void recognizesRacecarAsPalindrome() { }
```

### Format

```java
// Option 1: should + behavior
@Test
void shouldApplyTaxBasedOnShippingState() { }

// Option 2: when + then
@Test
void whenAdding2And3_thenReturns5() { }

// Option 3: Given-When-Then (for complex scenarios)
class PremiumUserCheckoutTest {
    @Test
    void givenPremiumUser_whenCheckout_thenReceives20PercentDiscount() { ... }
}
```

#### Preferred vocabulary
- Use "returns" instead of "yields"

---

## Test Doubles

### Dummy

Object passed but never used.

```java
Logger dummyLogger = new Logger() {
    @Override public void log(String message) { }
};
new UserService(realRepo, dummyLogger);
```

### Stub

Returns predefined values.

```java
UserRepo stubRepo = new UserRepo() {
    @Override public User findById(String id) {
        return new User("Test");
    }
    @Override public void save(User user) { }
};
```

### Spy

Records how it was called.

```java
class EmailSpy implements EmailService {
    final List<String> sentEmails = new ArrayList<>();

    @Override
    public void send(String to, String message) {
        sentEmails.add(to);
    }
}

// Later
assertTrue(emailSpy.sentEmails.contains("user@example.com"));
```

### Mock

Verifies expected interactions.

```java
// Using Mockito
@Mock UserRepo mockRepo;

@BeforeEach
void setUp() {
    MockitoAnnotations.openMocks(this);
    when(mockRepo.save(any())).thenReturn(null);
}

// After test
verify(mockRepo).save(expectedUser);
```

#### Variable naming
- Use "mock" instead of "stub"

### Fake

Working implementation (simplified).

```java
class InMemoryUserRepo implements UserRepo {
    private final Map<String, User> users = new HashMap<>();

    public void save(User user) {
        users.put(user.getId(), user);
    }

    public User findById(String id) {
        return users.get(id);
    }
}
```

---

## Testing Strategies by Layer

### Domain Layer (Most Tests)

- Unit tests with no mocks
- Test business rules, value objects, entities
- Fast, comprehensive

```java
class MoneyTest {
    @Test
    void addsAmountsWithSameCurrency() {
        Money a = Money.dollars(10);
        Money b = Money.dollars(20);
        assertEquals(Money.dollars(30), a.add(b));
    }

    @Test
    void throwsWhenAddingDifferentCurrencies() {
        Money usd = Money.dollars(10);
        Money eur = Money.euros(10);
        assertThrows(CurrencyMismatchException.class, () -> usd.add(eur));
    }
}
```

### Application Layer

- Integration tests with mocked infrastructure
- Test use case orchestration

```java
class CreateOrderUseCaseTest {
    @Test
    void createsOrderAndSendsConfirmation() {
        InMemoryOrderRepo orderRepo = new InMemoryOrderRepo();
        EmailService emailService = mock(EmailService.class);
        CreateOrderUseCase useCase = new CreateOrderUseCase(orderRepo, emailService);

        useCase.execute(new CreateOrderRequest(new CustomerId("123"), List.of(...)));

        assertEquals(1, orderRepo.count());
        verify(emailService).send(anyString(), anyString());
    }
}
```

### Infrastructure Layer

- Integration tests with real dependencies
- Test database, API integrations

```java
class PostgresOrderRepoTest {
    private PostgresOrderRepo repo;

    @BeforeAll
    void setUp() {
        repo = new PostgresOrderRepo(testDb);
    }

    @Test
    void persistsAndRetrievesOrder() {
        Order order = Order.create(...);
        repo.save(order);

        Order found = repo.findById(order.getId());
        assertEquals(order, found);
    }
}
```

---

## High-Value Integration Tests

Focus integration tests on:

1. **Boundaries** - Where systems meet
2. **Critical paths** - Money, security, core features
3. **Complex queries** - Database operations

### Contract Tests

Verify implementations match interfaces.

```java
// Shared contract test using abstract class
abstract class UserRepoContractTest {
    abstract UserRepo createRepo();
    
    private UserRepo repo;

    @BeforeEach
    void setUp() {
        repo = createRepo();
    }

    @Test
    void savesAndRetrievesUser() {
        User user = User.create(new Name("Test"));
        repo.save(user);
        User found = repo.findById(user.getId());
        assertEquals(user, found);
    }

    @Test
    void returnsNullForMissingUser() {
        User found = repo.findById(new UserId("nonexistent"));
        assertNull(found);
    }
}

// Apply to all implementations
class InMemoryUserRepoTest extends UserRepoContractTest {
    UserRepo createRepo() { return new InMemoryUserRepo(); }
}

class PostgresUserRepoTest extends UserRepoContractTest {
    UserRepo createRepo() { return new PostgresUserRepo(testDb); }
}
```

---

## Test Builders

Create test objects easily.

```java
class OrderBuilder {
    private OrderId id = new OrderId("order-1");
    private CustomerId customerId = new CustomerId("cust-1");
    private List<Item> items = new ArrayList<>();
    private OrderStatus status = OrderStatus.PENDING;

    OrderBuilder withId(OrderId id) {
        this.id = id;
        return this;
    }

    OrderBuilder withItems(List<Item> items) {
        this.items = items;
        return this;
    }

    OrderBuilder paid() {
        this.status = OrderStatus.PAID;
        return this;
    }

    Order build() {
        return Order.create(id, customerId, items, status);
    }
}

// Usage
Order order = new OrderBuilder()
    .withItems(List.of(new Item(new Sku("ABC"), Money.dollars(100))))
    .paid()
    .build();
```

---

## Common Testing Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Testing implementation | Brittle tests | Test behavior only |
| Too many mocks | Tests prove nothing | Use real objects when possible |
| Shared state | Flaky tests | Isolate each test |
| No assertions | False confidence | Always assert something meaningful |
| Testing trivial code | Wasted effort | Focus on logic and edge cases |
| Slow tests | Reduced feedback | Optimize, use unit tests |
| Mirroring implementation | Test becomes implementation | Verify outcomes, not algorithms |

### Tests Shouldn't Mirror Implementation Logic

When verifying behavior, don't reimplement the method's logic in the test. Tests should verify outcomes, not copy algorithms.

```java
// BAD: Mirrors implementation
@Test
void getAbsolutePath_returnsPath() {
    FileDirectory dir = new FileDirectory(tempDir);
    String result = dir.getAbsolutePath();

    // Reimplements exactly what the method does
    assertEquals(tempDir.toAbsolutePath().normalize().toString(), result);
}

// GOOD: Verifies outcome without copying logic
@Test
void getAbsolutePath_returnsAbsolutePath() {
    FileDirectory dir = new FileDirectory(tempDir);
    String absolutePath = dir.getAbsolutePath();

    // Verifies properties of the result
    assertNotNull(absolutePath);
    assertTrue(absolutePath.endsWith(tempDir.getFileName().toString()));
}
```

The first test breaks if the implementation changes how it builds the path. 
The second test verifies the actual requirement: that an absolute path is returned.
