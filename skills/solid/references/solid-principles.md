# SOLID Principles

## Overview

SOLID helps structure software to be flexible, maintainable, and testable. These principles reduce coupling and increase cohesion.

## S - Single Responsibility Principle (SRP)

> "A class should have one, and only one, reason to change."

### Problem It Solves
God objects that do everything - hard to test, hard to change, hard to understand.

### How to Apply
Each class handles ONE responsibility. If you find yourself saying "and" when describing what a class does, split it.

```java
// BAD: Multiple responsibilities
class Order {
    BigDecimal calculateTotal() { ... }
    void saveToDatabase() { ... }    // Persistence
    String generateInvoice() { ... } // Presentation
}

// GOOD: Single responsibility each
class Order {
    private List<OrderItem> items = new ArrayList<>();

    void addItem(OrderItem item) { ... }
    BigDecimal calculateTotal() { ... }
}

interface OrderRepository {
    void save(Order order);
}

class InvoiceGenerator {
    Invoice generate(Order order) { ... }
}
```

### Detection Questions
- Does this class have multiple reasons to change?
- Can I describe it without using "and"?
- Would different stakeholders request changes to different parts?

---

## O - Open/Closed Principle (OCP)

> "Software entities should be open for extension but closed for modification."

### Problem It Solves
Having to modify existing, tested code every time requirements change. Risk of breaking working features.

### How to Apply
Design abstractions that allow new behavior through new classes, not edits to existing ones.

```java
// BAD: Must modify to add new shipping
class ShippingCalculator {
    BigDecimal calculate(String type, BigDecimal value) {
        if ("standard".equals(type)) return value.compareTo(BigDecimal.valueOf(50)) < 0 ? BigDecimal.valueOf(5) : BigDecimal.ZERO;
        if ("express".equals(type)) return BigDecimal.valueOf(15);
        // Must add more ifs for new types!
        throw new IllegalArgumentException("Unknown type");
    }
}

// GOOD: Open for extension
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

// Add new shipping by creating new class, not modifying existing
class SameDayShipping implements ShippingMethod {
    public BigDecimal calculateCost(BigDecimal orderValue) {
        return BigDecimal.valueOf(25);
    }
}
```

### Architectural Insight
OCP at architecture level means: **design your codebase so new features are added by adding code, not changing existing code.**

---

## L - Liskov Substitution Principle (LSP)

> "Subtypes must be substitutable for their base types without altering program correctness."

### Problem It Solves
Subclasses that break expectations, requiring type-checking and special cases.

### How to Apply
Subclasses must honor the contract of the parent. If the parent returns positive numbers, subclasses cannot return negatives.

```java
// BAD: Violates parent's contract
abstract class DiscountPolicy {
    abstract BigDecimal getDiscount(BigDecimal value);
}

class WeirdDiscount extends DiscountPolicy {
    BigDecimal getDiscount(BigDecimal value) {
        return new BigDecimal("-5"); // Increases cost! Breaks expectations
    }
}

// GOOD: Enforces contract
abstract class DiscountPolicy {
    protected final BigDecimal discount;

    DiscountPolicy(BigDecimal discount) {
        if (discount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Discount must be non-negative");
        }
        this.discount = discount;
    }

    BigDecimal getDiscount() {
        return discount;
    }
}
```

### Key Insight
This is why you can swap `InMemoryUserRepo` for `PostgresUserRepo` - they both honor the `UserRepo` interface contract.

---

## I - Interface Segregation Principle (ISP)

> "Clients should not be forced to depend on methods they do not use."

### Problem It Solves
Fat interfaces that force partial implementations, empty methods, or throws.

### How to Apply
Split large interfaces into smaller, cohesive ones. Clients depend only on what they need.

```java
// BAD: Fat interface
interface WarehouseDevice {
    void printLabel(String orderId);
    String scanBarcode();
    void packageItem(String orderId);
}

class BasicPrinter implements WarehouseDevice {
    public void printLabel(String orderId) { /* works */ }
    public String scanBarcode() { throw new UnsupportedOperationException("Not supported"); } // Forced!
    public void packageItem(String orderId) { throw new UnsupportedOperationException("Not supported"); }
}

// GOOD: Segregated interfaces
interface LabelPrinter {
    void printLabel(String orderId);
}

interface BarcodeScanner {
    String scanBarcode();
}

interface ItemPackager {
    void packageItem(String orderId);
}

class BasicPrinter implements LabelPrinter {
    public void printLabel(String orderId) { /* only what it does */ }
}
```

### Detection
If you see `throw new Error("Not implemented")` or empty method bodies, the interface is too fat.

---

## D - Dependency Inversion Principle (DIP)

> "High-level modules should not depend on low-level modules. Both should depend on abstractions."

### Problem It Solves
Tight coupling to specific implementations (databases, APIs, frameworks). Hard to test, hard to swap.

### How to Apply
Depend on interfaces, inject implementations.

```java
// BAD: Direct dependency on concrete class
class OrderService {
    private final SendGridEmailService emailService = new SendGridEmailService(); // Locked in!

    void confirmOrder(String email) {
        emailService.send(email, "Order confirmed");
    }
}

// GOOD: Depend on abstraction
interface EmailService {
    void send(String to, String message);
}

class OrderService {
    private final EmailService emailService;

    OrderService(EmailService emailService) {
        this.emailService = emailService;
    }

    void confirmOrder(String email) {
        emailService.send(email, "Order confirmed");
    }
}

// Now can inject any implementation
new OrderService(new SendGridEmailService());
new OrderService(new SESEmailService());
new OrderService(new MockEmailService()); // For tests!
```

### The Dependency Rule
Source code dependencies should point **inward** toward high-level policies (domain logic), never toward low-level details (infrastructure).

```
Infrastructure → Application → Domain
      ↑              ↑            ↑
    (outer)       (middle)     (inner)

Dependencies flow: outer → inner
Never: inner → outer
```

---

## Applying SOLID at Architecture Level

These principles scale beyond classes:

| Principle | Architecture Application |
|-----------|--------------------------|
| SRP | Each bounded context has one responsibility |
| OCP | New features = new modules, not edits to existing |
| LSP | Microservices with same contract are substitutable |
| ISP | Thin interfaces between services |
| DIP | High-level business logic doesn't know about databases/frameworks |

---

## Quick Reference

| Principle | One-Liner | Red Flag |
|-----------|-----------|----------|
| SRP | One reason to change | "This class handles X and Y and Z" |
| OCP | Add, don't modify | `if/else` chains for types |
| LSP | Subtypes are substitutable | Type-checking in calling code |
| ISP | Small, focused interfaces | Empty method implementations |
| DIP | Depend on abstractions | `new ConcreteClass()` in business logic |

---

## Lessons from Practice

These lessons emerged from real refactoring and correct common mistakes made when applying SOLID mechanically.

### 1. Interfaces require a concrete reason (YAGNI beats speculative OCP)
Don't introduce an interface because the pattern "feels right". 
Ask: *is there more than one implementation now, or is one concretely planned?*
A single-implementation interface adds indirection without value. Wait until the second implementation exists — or is genuinely imminent.
Ask: *is another module going to use this interface?*
A single-module interface adds indirection without value. Wait until the second module exists — or is genuinely imminent.

### 2. Configuration belongs at the entry point, visibly
Constants like directory paths or environment settings belong in `Main` (or a dedicated config class), not buried inside 
domain objects. Hiding configuration inside a class makes the program harder to understand at a glance and gives that 
class a second reason to change.

### 3. `System.exit` belongs only in `Main`
Domain and infrastructure classes signal failure by *throwing*. Deciding what to do about it — exit, retry, log — is the 
application shell's responsibility. `System.exit` anywhere except `Main` is a code smell and makes classes untestable.

### 4. Constructor arguments vs. internal constants
If a value varies per use (a path, a label, a threshold), it belongs in the constructor — it is configuration the caller 
owns. If it is truly invariant and internal to the class, make it a private constant. The wrong split forces callers to 
pass things that are not their concern, or hides things that should be visible.

### 5. Abstraction level consistency within a method
Every statement in a method should operate at the same level of abstraction. If `main()` mixes high-level wiring 
(`new EdfParser()`) with low-level detail (`if (!dir.isDirectory())`), extract the detail into the class that owns that 
concern — not into a private helper that stays in `main`.

### 6. Don't sort where you don't own the ordering decision (SRP)
A class that provides access to a collection should not impose ordering unless that ordering is its core responsibility. 
Sorting alphabetically in `FileDirectory.listFiles()` violated SRP - the directory shouldn't decide how files are ordered. 
Return in natural order and let consumers sort by their criteria (date, size, name, etc.).
