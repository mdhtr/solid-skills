# Design Patterns

## What Are Design Patterns?

Reusable solutions to common design problems. A shared vocabulary for discussing design.

## WARNING: Don't Force Patterns

> "Let patterns emerge from refactoring, don't force them upfront."

Patterns should solve problems you HAVE, not problems you MIGHT have.

## When to Use Patterns

1. **You recognize the problem** - You've seen it before
2. **The pattern fits** - Not forcing it
3. **It simplifies** - Doesn't add unnecessary complexity
4. **Team understands it** - Shared knowledge

---

## Creational Patterns

### Singleton

**Purpose:** Ensure only one instance exists.

**When to use:** Global configuration, connection pools, logging.

**Warning:** Often overused. Consider dependency injection instead.

```java
class Logger {
    private static final Logger INSTANCE = new Logger();

    private Logger() {}

    static Logger getInstance() {
        return INSTANCE;
    }

    void log(String message) { ... }
}
```

### Factory

**Purpose:** Create objects without specifying exact class.

**When to use:** Object creation logic is complex, or varies by type.

```java
interface Notification {
    void send(String message);
}

class EmailNotification implements Notification { ... }
class SMSNotification implements Notification { ... }
class PushNotification implements Notification { ... }

class NotificationFactory {
    Notification create(String type) {
        return switch (type) {
            case "email" -> new EmailNotification();
            case "sms" -> new SMSNotification();
            case "push" -> new PushNotification();
            default -> throw new IllegalArgumentException("Unknown type");
        };
    }
}
```

### Builder

**Purpose:** Construct complex objects step by step.

**When to use:** Objects with many optional parameters, test data creation.

```java
class UserBuilder {
    private String name;
    private String email;
    private int age;

    UserBuilder withName(String name) {
        this.name = name;
        return this;
    }

    UserBuilder withEmail(String email) {
        this.email = email;
        return this;
    }

    UserBuilder withAge(int age) {
        this.age = age;
        return this;
    }

    User build() {
        return new User(name, email, age);
    }
}

// Usage
User user = new UserBuilder()
    .withName("Alice")
    .withEmail("alice@example.com")
    .build();
```

### Prototype

**Purpose:** Create new objects by cloning existing ones.

**When to use:** Object creation is expensive, or you need copies with slight variations.

```java
interface Prototype<T> {
    T clone();
}

class Document implements Prototype<Document> {
    private final String title;
    private final String content;
    private final Metadata metadata;

    Document(String title, String content, Metadata metadata) {
        this.title = title;
        this.content = content;
        this.metadata = metadata;
    }

    @Override
    public Document clone() {
        return new Document(title, content, new Metadata(metadata));
    }
}
```

---

## Structural Patterns

### Adapter

**Purpose:** Make incompatible interfaces work together.

**When to use:** Integrating third-party libraries, legacy code.

```java
// Third-party library with different interface
class OldPaymentAPI {
    boolean makePayment(int cents) { ... }
}

// Our interface
interface PaymentGateway {
    ChargeResult charge(Money amount);
}

// Adapter
class OldPaymentAdapter implements PaymentGateway {
    private final OldPaymentAPI oldAPI;

    OldPaymentAdapter(OldPaymentAPI oldAPI) {
        this.oldAPI = oldAPI;
    }

    public ChargeResult charge(Money amount) {
        int cents = amount.toCents();
        boolean success = oldAPI.makePayment(cents);
        return success ? ChargeResult.success() : ChargeResult.failed();
    }
}
```

### Decorator

**Purpose:** Add behavior to objects dynamically.

**When to use:** Adding features without modifying existing code.

```java
interface Notifier {
    void send(String message);
}

class EmailNotifier implements Notifier {
    public void send(String message) {
        System.out.println("Email: " + message);
    }
}

// Decorators
class SMSDecorator implements Notifier {
    private final Notifier wrapped;

    SMSDecorator(Notifier wrapped) {
        this.wrapped = wrapped;
    }

    public void send(String message) {
        wrapped.send(message);
        System.out.println("SMS: " + message);
    }
}

class SlackDecorator implements Notifier {
    private final Notifier wrapped;

    SlackDecorator(Notifier wrapped) {
        this.wrapped = wrapped;
    }

    public void send(String message) {
        wrapped.send(message);
        System.out.println("Slack: " + message);
    }
}

// Usage - compose behaviors
Notifier notifier = new SlackDecorator(
    new SMSDecorator(
        new EmailNotifier()
    )
);
notifier.send("Alert!"); // Sends to all three
```

### Proxy

**Purpose:** Control access to an object.

**When to use:** Lazy loading, access control, logging, caching.

```java
interface Image {
    void display();
}

class RealImage implements Image {
    private final String filename;

    RealImage(String filename) {
        this.filename = filename;
        loadFromDisk(); // Expensive
    }

    private void loadFromDisk() { ... }

    public void display() { ... }
}

// Lazy loading proxy
class ImageProxy implements Image {
    private RealImage realImage;
    private final String filename;

    ImageProxy(String filename) {
        this.filename = filename;
    }

    public void display() {
        if (realImage == null) {
            realImage = new RealImage(filename);
        }
        realImage.display();
    }
}
```

### Composite

**Purpose:** Treat individual objects and compositions uniformly.

**When to use:** Tree structures, hierarchies (files/folders, UI components).

```java
interface Component {
    Money getPrice();
}

class Product implements Component {
    private final Money price;

    Product(Money price) {
        this.price = price;
    }

    public Money getPrice() {
        return price;
    }
}

class Box implements Component {
    private final List<Component> children = new ArrayList<>();

    void add(Component component) {
        children.add(component);
    }

    public Money getPrice() {
        return children.stream()
            .map(Component::getPrice)
            .reduce(Money.zero(), Money::add);
    }
}

// Usage
Box smallBox = new Box();
smallBox.add(new Product(Money.dollars(10)));
smallBox.add(new Product(Money.dollars(20)));

Box bigBox = new Box();
bigBox.add(smallBox);
bigBox.add(new Product(Money.dollars(50)));

System.out.println(bigBox.getPrice()); // $80.00
```

---

## Behavioral Patterns

### Strategy

**Purpose:** Define a family of algorithms, make them interchangeable.

**When to use:** Multiple ways to do something, switchable at runtime.

```java
interface PricingStrategy {
    Money calculate(Money basePrice);
}

class RegularPricing implements PricingStrategy {
    public Money calculate(Money basePrice) {
        return basePrice;
    }
}

class PremiumDiscount implements PricingStrategy {
    public Money calculate(Money basePrice) {
        return basePrice.multiply(0.8); // 20% off
    }
}

class BlackFriday implements PricingStrategy {
    public Money calculate(Money basePrice) {
        return basePrice.multiply(0.5); // 50% off
    }
}

class ShoppingCart {
    private final PricingStrategy pricing;

    ShoppingCart(PricingStrategy pricing) {
        this.pricing = pricing;
    }

    Money calculateTotal(List<Item> items) {
        Money base = items.stream()
            .map(Item::getPrice)
            .reduce(Money.zero(), Money::add);
        return pricing.calculate(base);
    }
}
```

### Observer

**Purpose:** Notify multiple objects about state changes.

**When to use:** Event systems, pub/sub, reactive updates.

```java
interface Observer {
    void update(Event event);
}

class EventEmitter {
    private final List<Observer> observers = new ArrayList<>();

    void subscribe(Observer observer) {
        observers.add(observer);
    }

    void unsubscribe(Observer observer) {
        observers.remove(observer);
    }

    void notify(Event event) {
        for (Observer o : observers) {
            o.update(event);
        }
    }
}

// Usage
class OrderService extends EventEmitter {
    void placeOrder(Order order) {
        // Process order...
        notify(new Event("ORDER_PLACED", order));
    }
}

class EmailService implements Observer {
    public void update(Event event) {
        if ("ORDER_PLACED".equals(event.getType())) {
            sendConfirmation(event.getOrder());
        }
    }
}
```

### Template Method

**Purpose:** Define algorithm skeleton, let subclasses override steps.

**When to use:** Common algorithm with varying steps.

```java
abstract class DataExporter {
    // Template method - defines the algorithm
    void export(List<Data> data) {
        validate(data);
        String formatted = format(data);
        write(formatted);
        notify();
    }

    // Common steps
    private void validate(List<Data> data) { ... }
    private void notify() { ... }

    // Steps to override
    protected abstract String format(List<Data> data);
    protected abstract void write(String content);
}

class CSVExporter extends DataExporter {
    protected String format(List<Data> data) {
        return data.stream()
            .map(Data::toCSV)
            .collect(Collectors.joining("\n"));
    }

    protected void write(String content) {
        // Write to export.csv
    }
}

class JSONExporter extends DataExporter {
    protected String format(List<Data> data) {
        // Convert to JSON
        return "...";
    }

    protected void write(String content) {
        // Write to export.json
    }
}
```

### Command

**Purpose:** Encapsulate a request as an object.

**When to use:** Undo/redo, queuing, logging actions.

```java
interface Command {
    void execute();
    void undo();
}

class AddItemCommand implements Command {
    private final Cart cart;
    private final Item item;

    AddItemCommand(Cart cart, Item item) {
        this.cart = cart;
        this.item = item;
    }

    public void execute() {
        cart.add(item);
    }

    public void undo() {
        cart.remove(item);
    }
}

class CommandHistory {
    private final Deque<Command> history = new ArrayDeque<>();

    void execute(Command command) {
        command.execute();
        history.push(command);
    }

    void undo() {
        Command command = history.poll();
        if (command != null) {
            command.undo();
        }
    }
}
```

---

## Pattern Awareness

### The Four-Dimensional Lens

When analyzing new code/libraries, ask:

1. **What problem does it solve?** (Creational, Structural, Behavioral)
2. **What scope?** (Object-level, Class-level, System-level)
3. **When is it applied?** (Compile-time, Runtime)
4. **How coupled?** (Tight, Loose)

This helps recognize patterns even in unfamiliar code.

---

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| **God Object** | Class does everything | Split by responsibility |
| **Spaghetti Code** | Tangled, no structure | Refactor to layers |
| **Golden Hammer** | Using one pattern for everything | Match pattern to problem |
| **Premature Optimization** | Optimizing before needed | YAGNI, profile first |
| **Copy-Paste Programming** | Duplication | Extract, Rule of Three |
