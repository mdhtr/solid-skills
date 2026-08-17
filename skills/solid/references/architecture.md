# Software Architecture

## The Goal of Architecture

Enable the development team to:
1. **Add** features with minimal friction
2. **Change** existing features safely
3. **Remove** features cleanly
4. **Test** features in isolation
5. **Deploy** independently when possible

## Architectural Principles

### 1. Vertical Boundaries (Features/Slices)

Organize by **feature**, not by technical layer.

```
BAD: Layer-first
src/
  controllers/
    UserController.java
    OrderController.java
  services/
    UserService.java
    OrderService.java
  repositories/
    UserRepository.java
    OrderRepository.java

GOOD: Feature-first
src/
  users/
    UserController.java
    UserService.java
    UserRepository.java
  orders/
    OrderController.java
    OrderService.java
    OrderRepository.java
```

**Why:** Changes to "users" feature stay in `users/`. High cohesion within features.

### 2. Horizontal Boundaries (Layers)

Separate concerns into layers with clear dependencies.

```
┌──────────────────────────────────────┐
│           Presentation               │  UI, Controllers, CLI
├──────────────────────────────────────┤
│           Application                │  Use Cases, Orchestration
├──────────────────────────────────────┤
│             Domain                   │  Business Logic, Entities
├──────────────────────────────────────┤
│          Infrastructure              │  Database, APIs, External
└──────────────────────────────────────┘
```

### 3. The Dependency Rule

**Dependencies point INWARD.**

```
Infrastructure → Application → Domain
      ↓               ↓            ↓
   (outer)        (middle)      (inner)
```

- Inner layers know NOTHING about outer layers
- Domain has zero dependencies on infrastructure
- Use interfaces to invert dependencies

```java
// Domain defines the interface (inner)
interface UserRepository {
    void save(User user);
    User findById(UserId id);
}

// Infrastructure implements it (outer)
class PostgresUserRepository implements UserRepository {
    public void save(User user) {
        // SQL here
    }
    public User findById(UserId id) {
        // SQL here
    }
}

// Domain service uses the interface
class UserService {
    private final UserRepository repo;
    UserService(UserRepository repo) { // Depends on abstraction
        this.repo = repo;
    }
}
```

### 4. Contracts

Interfaces define boundaries between components.

```java
// The contract
interface PaymentGateway {
    ChargeResult charge(Money amount, CardDetails card);
    RefundResult refund(String chargeId);
}

// Multiple implementations possible
class StripeGateway implements PaymentGateway { }
class PayPalGateway implements PaymentGateway { }
class MockGateway implements PaymentGateway { }  // For tests
```

### 5. Cross-Cutting Concerns

Concerns that span multiple features: logging, auth, validation, error handling.

**Options:**
- Middleware/interceptors
- Decorators
- Aspect-oriented approaches
- Base classes (use sparingly)

```java
// Middleware approach
class LoggingMiddleware {
    Response handle(Request request, Handler next) {
        System.out.println("Request: " + request.getPath());
        Response response = next.handle(request);
        System.out.println("Response: " + response.getStatus());
        return response;
    }
}
```

### 6. Conway's Law

> "Organizations design systems that mirror their communication structure."

**Implication:** Team structure affects architecture. Align both intentionally.

---

## Common Architectural Styles

### Layered Architecture

Traditional layers: Presentation → Business → Persistence

**Pros:** Simple, well-understood
**Cons:** Can become a "big ball of mud" without discipline

### Hexagonal Architecture (Ports & Adapters)

Domain at center, adapters around the edges.

```
        ┌─────────────────────┐
        │     HTTP Adapter    │
        └─────────┬───────────┘
                  │
┌─────────────────▼─────────────────┐
│              DOMAIN                │
│   ┌─────────────────────────┐     │
│   │      Business Logic      │     │
│   │      Use Cases           │     │
│   └─────────────────────────┘     │
└─────────────────┬─────────────────┘
                  │
        ┌─────────▼───────────┐
        │   Database Adapter   │
        └─────────────────────┘
```

**Ports:** Interfaces defined by the domain
**Adapters:** Implementations that connect to the outside world

### Clean Architecture

Similar to Hexagonal, with explicit layers:

1. **Entities** - Enterprise business rules
2. **Use Cases** - Application business rules
3. **Interface Adapters** - Controllers, Presenters, Gateways
4. **Frameworks & Drivers** - Web, DB, External interfaces

---

## Feature-Driven Structure (Frontend)

```
src/
  features/
    auth/
      components/
        LoginForm.java
        SignupForm.java
      services/
        AuthService.java
      dto/
        AuthRequest.java
        AuthResponse.java
      index.java  # Public API
    checkout/
      components/
      services/
      dto/
      index.java
  shared/
    components/  # Truly shared UI
    services/    # Truly shared services
    utils/       # Truly shared utilities
```

---

## Feature-Driven Structure (Backend)

```
src/
  modules/
    users/
      domain/
        User.java
        UserRepository.java  # Interface
      application/
        CreateUser.java      # Use case
        GetUser.java         # Use case
      infrastructure/
        PostgresUserRepo.java
      presentation/
        UserController.java
        UserDTO.java
    orders/
      domain/
      application/
      infrastructure/
      presentation/
  shared/
    domain/        # Shared value objects
    infrastructure/ # Shared infra utilities
```

---

## The Walking Skeleton

Start with a minimal end-to-end slice:

1. **Thinnest possible feature** that touches all layers
2. **Deployable** from day one
3. **Proves the architecture** works

Example walking skeleton for e-commerce:
- User can view ONE product (hardcoded)
- User can add it to cart
- User can "checkout" (just logs)

From there, flesh out each feature fully.

---

## Testing Architecture

```
┌────────────────────────────────────────────┐
│            E2E / Acceptance Tests          │  Few, slow, high confidence
├────────────────────────────────────────────┤
│            Integration Tests               │  Some, medium speed
├────────────────────────────────────────────┤
│              Unit Tests                    │  Many, fast, isolated
└────────────────────────────────────────────┘
```

**Test by layer:**
- **Domain:** Unit tests (most tests here)
- **Application:** Integration tests with mocked infra
- **Infrastructure:** Integration tests with real dependencies
- **E2E:** Critical paths only

---

## Architecture Decision Records (ADRs)

Document significant decisions:

```markdown
# ADR 001: Use PostgreSQL for persistence

## Status
Accepted

## Context
We need a database. Options: PostgreSQL, MongoDB, MySQL

## Decision
PostgreSQL for:
- ACID compliance
- Team familiarity
- JSON support for flexibility

## Consequences
- Need PostgreSQL expertise
- Schema migrations required
- Excellent query capabilities
```

---

## Lessons from Practice

These lessons emerged from real refactoring and concern the application entry point and error handling boundaries.

### 1. Configuration belongs at the entry point, visibly
Constants like directory paths or environment settings belong in `Main` (or a dedicated config class), not buried inside
domain objects. Hiding configuration inside a class makes the program harder to understand at a glance and gives that
class a second reason to change. The entry point should be the single place where wiring configuration is visible.

### 2. `System.exit` belongs only in `Main`
Domain and infrastructure classes signal failure by *throwing*. Deciding what to do about it — exit, retry, log — is the
application shell's responsibility. `System.exit` anywhere except `Main` is a code smell and makes classes untestable.

---

## Red Flags in Architecture

- **Circular dependencies** between modules
- **Domain depending on infrastructure**
- **Framework code in business logic**
- **No clear boundaries** between features
- **Shared mutable state** across modules
- **"Util" or "Common" packages** that grow forever
- **Database schema driving domain model**
