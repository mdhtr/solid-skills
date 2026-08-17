# Object-Oriented Design

## Responsibility-Driven Design (RDD)

The key insight: **Objects are defined by their responsibilities, not their data.**

### Finding Objects

Start with:
1. **Nouns** in requirements → candidate objects
2. **Verbs** → candidate methods/behaviors
3. **Domain concepts** → value objects

### Finding Responsibilities

Each object should answer:
- What does this object **know**?
- What does this object **do**?
- What does this object **decide**?

### Object Stereotypes

Every class fits one (or maybe two) stereotypes:

| Stereotype | Purpose | Example |
|------------|---------|---------|
| **Information Holder** | Knows things, holds data | `User`, `Product`, `Address` |
| **Structurer** | Maintains relationships | `OrderItems`, `UserGroup` |
| **Service Provider** | Performs work | `PaymentProcessor`, `EmailSender` |
| **Coordinator** | Orchestrates workflow | `OrderFulfillmentService` |
| **Controller** | Makes decisions, delegates | `CheckoutController` |
| **Interfacer** | Transforms between systems | `UserAPIAdapter`, `DatabaseMapper` |

### The Two Questions

For every class, ask:
1. **"What pattern is this?"** - Which stereotype? Which design pattern?
2. **"Is it doing too much?"** - Check object calisthenics rules

If you can't answer clearly, the class needs refactoring.

### Lessons from practice

#### Put the detail where the concern lives
When a method needs low-level validation or mechanics, put that detail in the class that **owns the concern** — 
not in the caller, and not in a private helper that stays in the caller. The caller should delegate, not micromanage.

```java
// BAD: Caller owns validation detail it shouldn't know
Path path = Paths.get(DATA_DIR);
if (!Files.isDirectory(path)) {  // The caller shouldn't know how to validate paths
    throw new IllegalArgumentException("Not a directory");
}
FileDirectory dir = new FileDirectory(path);

// GOOD: Detail lives in the class that owns it
FileDirectory dir = new FileDirectory(Paths.get(DATA_DIR));  // Validation happens inside
```

---

## Tell, Don't Ask

**Command objects to do work. Don't interrogate them and do the work yourself.**

```java
// BAD: Asking, then doing
if (account.getBalance().compareTo(amount) >= 0) {
    account.setBalance(account.getBalance().subtract(amount));
    // more logic here...
}

// GOOD: Telling
WithdrawResult result = account.withdraw(amount);
if (result.isSuccess()) {
    // ...
}
```

The object that has the data should have the behavior.

---

## Design by Contract (DbC)

Every method has:
- **Preconditions** - What must be true BEFORE calling
- **Postconditions** - What will be true AFTER calling
- **Invariants** - What is ALWAYS true about the object

```java
class BankAccount {
    private Money balance;

    // INVARIANT: balance is never negative

    // PRECONDITION: amount > 0
    // POSTCONDITION: balance decreased by amount OR error returned
    WithdrawResult withdraw(Money amount) {
        if (amount.isNegativeOrZero()) {
            return WithdrawResult.invalidAmount();
        }

        if (balance.isLessThan(amount)) {
            return WithdrawResult.insufficientFunds();
        }

        balance = balance.subtract(amount);
        return WithdrawResult.success(balance);
    }
}
```

---

## Composition Over Inheritance

**Prefer composing objects over extending classes.**

### Why Inheritance is Problematic:
- Tight coupling between parent and child
- Fragile base class problem
- Difficult to change parent without breaking children
- Forces "is-a" relationship that may not fit

### When to Use Inheritance:
- True "is-a" relationship (rare)
- Framework requirements
- Template Method pattern (intentional)

### Prefer Composition:
```java
// BAD: Inheritance
class PremiumUser extends User {
    int getDiscount() { return 20; }
}

// GOOD: Composition
class User {
    private final DiscountPolicy discountPolicy;

    User(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }

    int getDiscount() {
        return discountPolicy.calculate();
    }
}

// Now discount behavior is pluggable
new User(new PremiumDiscount());
new User(new StandardDiscount());
new User(new NoDiscount());
```

---

## The Law of Demeter (Principle of Least Knowledge)

**Only talk to your immediate friends.**

A method should only call:
1. Methods on `this`
2. Methods on parameters
3. Methods on objects it creates
4. Methods on its direct components

```java
// BAD: Reaching through objects
order.getCustomer().getAddress().getCity();

// GOOD: Ask the immediate friend
order.getShippingCity();
```

This reduces coupling - changes to `Address` don't ripple through all callers.

---

## Encapsulation

**Hide internal details, expose behavior.**

### Levels of Encapsulation:
1. **Data** - private fields, no direct access
2. **Implementation** - how things work internally
3. **Type** - concrete class hidden behind interface
4. **Design** - architectural decisions hidden from clients

```java
// BAD: Exposed internals
class Order {
    public List<Item> items = new ArrayList<>();
    public BigDecimal total = BigDecimal.ZERO;
}

// Client can corrupt state
order.items.add(item);
order.total = new BigDecimal("-999"); // Oops!

// GOOD: Encapsulated
class Order {
    private OrderItems items;
    private Money total;

    void addItem(Item item) {
        items.add(item);
        recalculateTotal();
    }

    Money getTotal() {
        return total; // Returns copy or immutable
    }
}
```

---

## Polymorphism

**Replace conditionals with types.**

```java
// BAD: Type checking
BigDecimal calculateShipping(String method, BigDecimal value) {
    if ("standard".equals(method)) return value.compareTo(BigDecimal.valueOf(50)) < 0 ? BigDecimal.valueOf(5) : BigDecimal.ZERO;
    if ("express".equals(method)) return BigDecimal.valueOf(15);
    if ("overnight".equals(method)) return BigDecimal.valueOf(25);
    throw new IllegalArgumentException("Unknown method");
}

// GOOD: Polymorphism
interface ShippingMethod {
    BigDecimal calculateCost(BigDecimal orderValue);
}

class StandardShipping implements ShippingMethod {
    public BigDecimal calculateCost(BigDecimal orderValue) {
        return orderValue.compareTo(BigDecimal.valueOf(50)) < 0 ? BigDecimal.valueOf(5) : BigDecimal.ZERO;
    }
}

class ExpressShipping implements ShippingMethod {
    public BigDecimal calculateCost(BigDecimal orderValue) {
        return BigDecimal.valueOf(15);
    }
}

// Usage - no conditionals
BigDecimal calculateShipping(ShippingMethod method, BigDecimal value) {
    return method.calculateCost(value);
}
```

---

## Value Objects vs Entities

### Value Objects
- Defined by their attributes (no identity)
- Immutable
- Comparable by value
- Examples: `Money`, `Email`, `Address`, `DateRange`

```java
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof Money)) return false;
        Money other = (Money) obj;
        return amount.equals(other.amount) &&
               currency.equals(other.currency);
    }

    public Money add(Money other) {
        if (!currency.equals(other.currency)) {
            throw new CurrencyMismatchException();
        }
        return new Money(amount.add(other.amount), currency);
    }
}
```

### Entities
- Have identity (survives attribute changes)
- Usually mutable (via methods)
- Comparable by identity
- Examples: `User`, `Order`, `Product`

```java
public class User {
    private final UserId id;
    private Email email;
    private Name name;

    public User(UserId id, Email email, Name name) {
        this.id = id;
        this.email = email;
        this.name = name;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof User)) return false;
        User other = (User) obj;
        return id.equals(other.id); // Identity comparison
    }

    public void changeEmail(Email newEmail) {
        this.email = newEmail; // Still same user
    }
}
```

---

## Aggregates

A cluster of objects treated as a single unit for data changes.

- One object is the **aggregate root** (entry point)
- External code only references the root
- Root enforces invariants for the entire cluster

```java
// Order is the aggregate root
class Order {
    private List<OrderItem> items = new ArrayList<>();

    // All access through the root
    void addItem(Product product, int quantity) {
        OrderItem item = new OrderItem(product, quantity);
        items.add(item);
        validateTotal();
    }

    void removeItem(ItemId itemId) {
        items.removeIf(i -> !i.getId().equals(itemId));
    }

    // Root enforces invariants
    private void validateTotal() {
        if (calculateTotal().exceeds(MAX_ORDER_VALUE)) {
            throw new OrderTotalExceededException();
        }
    }
}

// BAD: Accessing items directly
order.items.add(new OrderItem(...)); // Bypasses validation!

// GOOD: Through the root
order.addItem(product, 2); // Validation happens
```

---

## API Design Lessons

### Accept modern types, let caller handle conversion
Design public APIs with modern types (`Path`, `Instant`) to clearly signal what you expect. Let the caller handle 
conversion from primitive strings, managing parsing exceptions (`InvalidPathException`) as their own concern. 
This separates path parsing from domain validation.

```java
// GOOD: Accept Path, caller converts from String

// Caller handles path parsing
public static void main(String[] args) {
    try {
        Path dataPath = Paths.get(DATA_DIR);  // Caller owns conversion
        FileDirectory directory = new FileDirectory(dataPath);
    } catch (InvalidPathException e) {
        System.err.println("Invalid path: " + e.getMessage());
        System.exit(1);
    }
}

// Class accepts validated Path
class FileDirectory {
    FileDirectory(Path path) {
        this.directory = path.toAbsolutePath().normalize().toFile();
        // Only validates directory existence, not path syntax
    }
}
```

### Distinguish error causes with specific messages
Different failure modes deserve different error messages. If you can distinguish "path doesn't exist" from "path is a file", 
do so. Package-private constants (`ERRORMESSAGE_PATH_DOES_NOT_EXIST`) shared with tests prevent message drift while 
maintaining the specific distinction users need for debugging.

```java
static final String ERRORMESSAGE_PATH_DOES_NOT_EXIST = "Path does not exist";
static final String ERRORMESSAGE_PATH_NOT_A_DIRECTORY = "Path is not a directory";

public FileDirectory(Path path) {
    if (!directory.exists()) {
        throw new IllegalArgumentException(ERRORMESSAGE_PATH_DOES_NOT_EXIST + ": " + path);
    }
    if (!directory.isDirectory()) {
        throw new IllegalArgumentException(ERRORMESSAGE_PATH_NOT_A_DIRECTORY + ": " + path);
    }
}
```

### Fail-fast in constructors for initialization-time failures
Constructor validation is appropriate when an invalid argument prevents the program from running. A missing directory
is a fatal config error at startup, not optional behavior. Use `try-catch` in `main()` for these, not `Optional` return
types that suggest the caller can proceed without the object.

### Constructor arguments vs. internal constants
If a value varies per use (a path, a label, a threshold), it belongs in the constructor — it is configuration the caller
owns. If it is truly invariant and internal to the class, make it a private constant. The wrong split forces callers to
pass things that are not their concern, or hides things that should be visible.
