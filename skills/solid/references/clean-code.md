# Clean Code Practices

## What is Clean Code?

Code that is:
- **Easy to understand** - reveals intent clearly
- **Easy to change** - modifications are localized
- **Easy to test** - dependencies are injectable
- **Simple** - no unnecessary complexity

## The Human-Centered Approach

Code has THREE consumers:
1. **Users** - get their needs met
2. **Customers** - make or save money
3. **Developers** - must maintain it

Design for all three, but remember: **developers read code 10x more than they write it.**

## Naming Principles

### 1. Consistency & Uniqueness (HIGHEST PRIORITY)
Same concept = same name everywhere. One name per concept.

```java
// BAD: Inconsistent names for same concept
getUserById(id);
fetchCustomerById(id);
retrieveClientById(id);

// GOOD: Consistent
getUser(id);
getOrder(id);
getProduct(id);
```

### 2. Understandability
Use domain language, not technical jargon.

```java
// BAD: Technical
var arr = users.stream().filter(u -> u.isActive()).toList();

// GOOD: Domain language
var activeCustomers = users.stream().filter(user -> user.isActive()).toList();
```

### 3. Specificity
Avoid vague names: `data`, `info`, `manager`, `handler`, `processor`, `utils`

```java
// BAD: Vague
class DataManager { }
void processInfo(Object data) { }

// GOOD: Specific
class OrderRepository { }
void validatePayment(Payment payment) { }
```

### 4. Brevity (but not at cost of clarity)
Short names are good only if meaning is preserved.

```java
// BAD: Too cryptic
var usrLst = getUsrs();

// BAD: Unnecessarily long
var listOfAllActiveUsersInTheSystem = getActiveUsers();

// GOOD: Brief but clear
var activeUsers = getActiveUsers();
```

### 5. Searchability
Names should be unique enough to grep/search.

```java
// BAD: Common word, hard to search
var data = fetch();

// GOOD: Unique, searchable
var orderSummary = fetchOrderSummary();
```

### 6. Pronounceability
You should be able to say it in conversation.

```java
// BAD
var genymdhms = generateYearMonthDayHourMinuteSecond();

// GOOD
var timestamp = generateTimestamp();
```

### 7. Austerity
Avoid unnecessary filler words.

```java
// BAD: Redundant
var userData = user; // 'Data' adds nothing
class UserClass { } // 'Class' adds nothing

// GOOD
var user = user;
class User { }
```

### 8. Name methods by outcome, not procedure
A method name should say *what* it returns or achieves, not *how* it works.

```java
// BAD: Describes the procedure
List<EdfFileInfo> parse() { ... }

// GOOD: Describes the outcome
List<EdfFileInfo> getFileInfos() { ... }
```

### 9. Methods should do one thing
If a name uses "and", it advertises two things — even if the implementation is short.

---

## Object Calisthenics (9 Rules)

Exercises to improve OO design. Follow strictly during practice, relax slightly in production.

### 1. One Level of Indentation per Method

**Detect mixed abstraction:**
- High-level intent (`processOrder`) mixed with low-level mechanics (`file.getName().endsWith(".txt")`)
- Object wiring (`new Service()`) mixed with validation (`if (!dir.isDirectory())`)
- Domain language (`calculateTotal`) mixed with implementation details (`stream().filter().map().collect()`)

**Fix:** Extract methods so every statement operates at the same conceptual level.

```java
// BAD: Multiple levels
void process(List<Order> orders) {
    for (Order order : orders) {
        if (order.isValid()) {
            for (OrderItem item : order.getItems()) {
                if (item.isInStock()) {
                    // process...
                }
            }
        }
    }
}

// GOOD: Extract methods
void process(List<Order> orders) {
    orders.stream()
          .filter(Order::isValid)
          .forEach(this::processOrder);
}

void processOrder(Order order) {
    order.getItems().stream()
         .filter(OrderItem::isInStock)
         .forEach(this::processItem);
}
```

### 2. Don't Use the ELSE Keyword

Use early returns, guard clauses, or polymorphism.

```java
// BAD: else
int getDiscount(User user) {
    if (user.isPremium()) {
        return 20;
    } else {
        return 0;
    }
}

// GOOD: Early return
int getDiscount(User user) {
    if (user.isPremium()) return 20;
    return 0;
}
```

### 3. Wrap All Primitives and Strings

Primitives should be wrapped in domain objects when they have meaning.

```java
// BAD: Primitive obsession
void createUser(String email, int age) { }

// GOOD: Value objects
public final class Email {
    private final String value;
    public Email(String value) {
        if (!isValid(value)) throw new IllegalArgumentException("Invalid email");
        this.value = value;
    }
    private boolean isValid(String email) { return email.contains("@"); }
    public String getValue() { return value; }
}

public final class Age {
    private final int value;
    public Age(int value) {
        if (value < 0 || value > 150) throw new IllegalArgumentException("Invalid age");
        this.value = value;
    }
    public int getValue() { return value; }
}

void createUser(Email email, Age age) { }
```

### 4. First-Class Collections

Any class with a collection should have no other instance variables.

```java
// BAD: Collection mixed with other state
class Order {
    List<OrderItem> items = new ArrayList<>();
    String customerId;
    BigDecimal total;
}

// GOOD: Collection is its own class
class OrderItems {
    private final List<OrderItem> items;
    OrderItems(List<OrderItem> items) { this.items = new ArrayList<>(items); }
    void add(OrderItem item) { ... }
    Money total() { ... }
    boolean isEmpty() { return items.isEmpty(); }
}

class Order {
    private final OrderItems items;
    private final CustomerId customerId;
    Order(OrderItems items, CustomerId customerId) {
        this.items = items;
        this.customerId = customerId;
    }
}
```

### 5. One Dot per Line (Law of Demeter)

Don't chain through object graphs.

```java
// BAD: Train wreck
var city = order.getCustomer().getAddress().getCity();

// GOOD: Tell, don't ask
var city = order.getShippingCity();
```

### 6. Don't Abbreviate

If a name is too long to type, the class is doing too much.

```java
// BAD
var custRepo = new CustRepo();
var ord = new Ord();

// GOOD
var customerRepository = new CustomerRepository();
var order = new Order();
```

### 7. Keep All Entities Small

- Classes: < 50 lines
- Methods: < 10 lines
- Files: < 100 lines

If larger, it's probably doing too much. Split it.

### 8. No Classes with More Than Two Instance Variables

Forces small, focused classes.

```java
// BAD: Too many variables
class Order {
    String id;
    String customerId;
    List<Item> items;
    BigDecimal total;
    String status;
}

// GOOD: Composed of smaller objects
class Order {
    private final OrderId id;
    private final OrderDetails details;
    Order(OrderId id, OrderDetails details) {
        this.id = id;
        this.details = details;
    }
}

class OrderDetails {
    private final Customer customer;
    private final LineItems lineItems;
    OrderDetails(Customer customer, LineItems lineItems) {
        this.customer = customer;
        this.lineItems = lineItems;
    }
}
```

### 9. No Getters/Setters/Properties

Objects should have behavior, not just data. Tell objects what to do.

```java
// BAD: Data bag with getters
class Account {
    BigDecimal getBalance() { return balance; }
    void setBalance(BigDecimal value) { this.balance = value; }
}

// Caller does the work
if (account.getBalance().compareTo(amount) >= 0) {
    account.setBalance(account.getBalance().subtract(amount));
}

// GOOD: Behavior-rich object
class Account {
    WithdrawResult withdraw(Money amount) {
        if (!canWithdraw(amount)) {
            return WithdrawResult.insufficientFunds();
        }
        this.balance = this.balance.subtract(amount);
        return WithdrawResult.success();
    }
}

// Caller tells, object decides
WithdrawResult result = account.withdraw(amount);
```

---

## Comments

### When to Write Comments

**Only write comments to explain WHY, not WHAT or HOW.**

Code explains what and how. Comments explain business reasons, non-obvious decisions, or warnings.

```java
// BAD: Explains what (redundant)
// Add 1 to counter
counter++;

// GOOD: Explains why
// Compensate for 0-based indexing in legacy API
counter++;
```

### Prefer Self-Documenting Code

Instead of commenting, rename to make intent clear.

```java
// BAD: Comment needed
// Check if user can access premium features
if (user.getSubscriptionLevel() >= 2 && !user.isBanned()) { }

// GOOD: Self-documenting
if (user.canAccessPremiumFeatures()) { }
```

---

## Formatting

### Vertical Spacing
- Related code together
- Blank lines between concepts
- Most important/public at top

### Horizontal Spacing
- Consistent indentation
- Space around operators
- Max line length ~80-120 characters

### Storytelling
Code should read top-to-bottom like a story. High-level at top, details below.

```java
class OrderProcessor {
    // Public API first
    ProcessResult process(Order order) {
        validate(order);
        calculateTotals(order);
        return save(order);
    }

    // Supporting methods below, in order of appearance
    private void validate(Order order) { ... }
    private void calculateTotals(Order order) { ... }
    private ProcessResult save(Order order) { ... }
}
```

---

## Type Clarity

### Use explicit types over `var` when type isn't obvious

Use `List<File> files = fileDirectory.listFiles()` instead of `var files = ...` when the right-hand side type isn't 
immediately visible from the expression. Reserve `var` for cases where the type is obvious from the literal.

```java
// GOOD: Type obvious from literal
var name = "Alice";
var count = 0;
var price = new BigDecimal("100.00");

// BAD: Type not obvious
var files = fileDirectory.listFiles();           // What type?
var result = service.calculateSomething(data);   // What type?

// GOOD: Explicit types when not obvious
List<File> files = fileDirectory.listFiles();
CalculationResult result = service.calculateSomething(data);
```

Explicit types improve readability, especially in tests where clarity matters most. The reader shouldn't have to 
infer types or jump to method definitions to understand variable types.
