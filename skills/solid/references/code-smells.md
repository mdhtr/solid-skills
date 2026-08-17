# Code Smells & Anti-Patterns

## What Are Code Smells?

Indicators that something MAY be wrong. Not bugs, but design problems that make code hard to understand, change, or test.

## The Five Categories

### 1. Bloaters
Code that has grown too large.

| Smell | Symptom | Refactoring |
|-------|---------|-------------|
| **Long Method** | > 10 lines | Extract Method |
| **Large Class** | > 50 lines, multiple responsibilities | Extract Class |
| **Long Parameter List** | > 3 parameters | Introduce Parameter Object |
| **Data Clumps** | Same group of variables appear together | Extract Class |
| **Primitive Obsession** | Primitives instead of small objects | Wrap in Value Object |

### 2. Object-Orientation Abusers
Misuse of OO principles.

| Smell | Symptom | Refactoring |
|-------|---------|-------------|
| **Switch Statements** | Type checking, large switch/if-else | Replace with Polymorphism |
| **Parallel Inheritance** | Adding subclass requires adding another | Merge Hierarchies |
| **Refused Bequest** | Subclass doesn't use parent methods | Replace Inheritance with Delegation |
| **Alternative Classes** | Different interfaces, same concept | Rename, Extract Superclass |

### 3. Change Preventers
Code that makes changes difficult.

| Smell | Symptom | Refactoring |
|-------|---------|-------------|
| **Divergent Change** | One class changed for many reasons | Extract Class (SRP) |
| **Shotgun Surgery** | One change touches many classes | Move Method/Field together |
| **Parallel Inheritance** | (see above) | Merge Hierarchies |

### 4. Dispensables
Code that can be removed.

| Smell | Symptom | Refactoring |
|-------|---------|-------------|
| **Comments** | Explaining bad code | Rename, Extract Method |
| **Duplicate Code** | Copy-paste | Extract Method, Pull Up Method |
| **Dead Code** | Unreachable code | Delete |
| **Speculative Generality** | "Just in case" code | Delete (YAGNI) |
| **Lazy Class** | Class that does almost nothing | Inline Class |

### 5. Couplers
Excessive coupling between classes.

| Smell | Symptom | Refactoring |
|-------|---------|-------------|
| **Feature Envy** | Method uses another class's data extensively | Move Method |
| **Inappropriate Intimacy** | Classes know too much about each other | Move Method, Extract Class |
| **Message Chains** | `a.getB().getC().getD()` | Hide Delegate |
| **Middle Man** | Class only delegates | Inline Class |

---

## The Seven Most Common Code Smells

### 1. Long Method

**Symptom:** Method > 10 lines, doing multiple things.

```java
// SMELL
void processOrder(Order order) {
    // Validate
    if (order.getItems().isEmpty()) throw new IllegalArgumentException("Empty");
    if (order.getCustomer() == null) throw new IllegalArgumentException("No customer");

    // Calculate
    Money total = Money.zero();
    for (OrderItem item : order.getItems()) {
        total = total.add(item.getPrice().multiply(item.getQuantity()));
        if (item.getDiscount() != null) {
            total = total.subtract(item.getDiscount());
        }
    }

    // Apply tax
    BigDecimal taxRate = getTaxRate(order.getCustomer().getState());
    total = total.add(total.multiply(taxRate));

    // Save
    db.orders.insert(order, total);

    // Notify
    emailService.send(order.getCustomer().getEmail(), "Order confirmed");
}

// REFACTORED
void processOrder(Order order) {
    validateOrder(order);
    Money total = calculateTotal(order);
    saveOrder(order, total);
    notifyCustomer(order);
}
```

### 2. Large Class

**Symptom:** Class with many responsibilities, > 50 lines.

```java
// SMELL: God class
class User {
    // User data
    String name;
    String email;

    // Authentication
    void login() { }
    void logout() { }
    void resetPassword() { }

    // Preferences
    void setTheme() { }
    void setLanguage() { }

    // Notifications
    void sendEmail() { }
    void sendSMS() { }

    // Billing
    void charge() { }
    void refund() { }
}

// REFACTORED: Separate classes
class User { String name; String email; }
class AuthService { void login(); void logout(); void resetPassword(); }
class UserPreferences { void setTheme(); void setLanguage(); }
class NotificationService { void sendEmail(); void sendSMS(); }
class BillingService { void charge(); void refund(); }
```

### 3. Feature Envy

**Symptom:** Method uses another class's data more than its own.

```java
// SMELL: Order envies Customer
class Order {
    Money calculateShipping(Customer customer) {
        if ("US".equals(customer.getCountry())) {
            if ("CA".equals(customer.getState())) return Money.dollars(10);
            return Money.dollars(15);
        }
        return Money.dollars(25);
    }
}

// REFACTORED: Move to Customer
class Customer {
    Money getShippingCost() {
        if ("US".equals(country)) {
            if ("CA".equals(state)) return Money.dollars(10);
            return Money.dollars(15);
        }
        return Money.dollars(25);
    }
}

class Order {
    Money calculateShipping() {
        return customer.getShippingCost();
    }
}
```

### 4. Primitive Obsession

**Symptom:** Using primitives for domain concepts.

```java
// SMELL
void createUser(String email, int age, String zipCode) {
    // No validation, easy to pass wrong values
    if (!email.contains("@")) throw new IllegalArgumentException();
    if (age < 0) throw new IllegalArgumentException();
}

// REFACTORED: Value objects
public final class Email {
    private final String value;
    public Email(String value) {
        if (!value.contains("@")) throw new IllegalArgumentException("Invalid email");
        this.value = value;
    }
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

void createUser(Email email, Age age, Address address) {
    // Type system prevents invalid data
}
```

### 5. Switch Statements

**Symptom:** Switching on type, repeated across codebase.

```java
// SMELL
double getArea(Shape shape) {
    switch (shape.getType()) {
        case "circle": return Math.PI * Math.pow(shape.getRadius(), 2);
        case "rectangle": return shape.getWidth() * shape.getHeight();
        case "triangle": return 0.5 * shape.getBase() * shape.getHeight();
    }
    return 0;
}

double getPerimeter(Shape shape) {
    switch (shape.getType()) { // Same switch again!
        case "circle": return 2 * Math.PI * shape.getRadius();
        // ...
    }
    return 0;
}

// REFACTORED: Polymorphism
interface Shape {
    double getArea();
    double getPerimeter();
}

class Circle implements Shape {
    private final double radius;
    Circle(double radius) { this.radius = radius; }
    public double getArea() { return Math.PI * radius * radius; }
    public double getPerimeter() { return 2 * Math.PI * radius; }
}
```

### 6. Inappropriate Intimacy

**Symptom:** Classes know too much about each other's internals.

```java
// SMELL
class Order {
    void process() {
        Inventory inventory = new Inventory();
        // Reaching into inventory's internals
        for (OrderItem item : items) {
            Stock stock = inventory.stockLevels.get(item.getSku());
            if (stock.getQuantity() < item.getQuantity()) {
                throw new IllegalStateException("Out of stock");
            }
            inventory.stockLevels.get(item.getSku()).deduct(item.getQuantity());
        }
    }
}

// REFACTORED: Tell, don't ask
class Inventory {
    ReserveResult reserve(List<OrderItem> items) {
        // Inventory manages its own state
        for (OrderItem item : items) {
            if (!canReserve(item)) {
                return ReserveResult.outOfStock(item);
            }
        }
        deductStock(items);
        return ReserveResult.success();
    }
}

class Order {
    void process(Inventory inventory) {
        ReserveResult result = inventory.reserve(items);
        if (!result.isSuccess()) {
            throw new OutOfStockException(result.getFailedItem());
        }
    }
}
```

### 7. Speculative Generality

**Symptom:** "Just in case" abstractions that aren't used.

```java
// SMELL: Over-engineered for hypothetical needs
interface PaymentProcessor {
    void process();
    void rollback();
    void audit();
    void generateReport();
    void scheduleRecurring();
}

class StripeProcessor implements PaymentProcessor {
    public void process() { /* actual code */ }
    public void rollback() { throw new UnsupportedOperationException("Not implemented"); }
    public void audit() { throw new UnsupportedOperationException("Not implemented"); }
    public void generateReport() { throw new UnsupportedOperationException("Not implemented"); }
    public void scheduleRecurring() { throw new UnsupportedOperationException("Not implemented"); }
}

// REFACTORED: YAGNI
interface PaymentProcessor {
    void process();
}

class StripeProcessor implements PaymentProcessor {
    public void process() { /* actual code */ }
}
// Add other methods when actually needed
```

---

## Prevention Strategies

1. **Follow Object Calisthenics** - Rules prevent most smells
2. **Practice TDD** - Tests reveal design problems early
3. **Review in pairs** - Fresh eyes catch smells
4. **Refactor continuously** - Don't let smells accumulate
5. **Apply SOLID** - Prevents structural smells
6. **Use static analysis** - Tools catch common issues

---

## When You Find a Smell

1. **Confirm it's a problem** - Not all smells need fixing
2. **Ensure test coverage** - Before refactoring
3. **Refactor in small steps** - Keep tests passing
4. **Commit frequently** - Easy to revert if needed
