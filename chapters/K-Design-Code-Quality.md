# K. Design & Code Quality Patterns (Complete + Principles + A-over-B Reasoning)

This chapter includes:

- ALL core software engineering principles (SOLID, DRY, WET, KISS, YAGNI, DAMP, GRASP, LoD)  
- Real-world examples  
- How to choose A over B  
- DDD + Hexagonal + Clean Architecture  
- Anti-patterns  
- Design patterns  
- Code review principles  
- Refactoring strategies  

---

# 1. SOFTWARE ENGINEERING PRINCIPLES (THE REAL FOUNDATION)

These are not “academic”—these are what **Staff/Principal engineers** actively use to guide architecture.

---

# 1.1 SOLID Principles (EXTREMELY DEEP & PRACTICAL)

## S — **Single Responsibility Principle (SRP)**  
*A class should have only one reason to change.*

### A-over-B decision:
✔ **Choose A (separate classes)** when:
- one class is handling more than one domain concept  
- one change touches multiple unrelated behaviors  

❌ Choose B (one class) only when:
- extremely simple domain  
- it’s accidental complexity to split  

### BAD:
```java
class OrderProcessor {
    void createOrder() {}
    void sendEmail() {}
    void pushToKafka() {}
    void reserveStock() {}
}
```

### GOOD:
```
OrderService
StockService
NotificationService
OrderEventPublisher
```

**Why A is better:**  
- easier testing  
- smaller blast radius  
- fewer merge conflicts  
- domain alignment  

---

## O — **Open/Closed Principle**  
*Open for extension, closed for modification.*

### A-over-B reasoning:
✔ Choose **Strategy Pattern** (A) over `if-else` chains (B) when:  
- new behaviors will be added frequently  
- you want pluggable logic  

### BAD:
```java
if(paymentType == "CARD")...
else if(paymentType == "CASH")...
else if(paymentType == "UPI")...
```

### GOOD:
```java
interface PaymentStrategy { pay(); }
class CardPayment implements PaymentStrategy {}
class UpiPayment implements PaymentStrategy {}
```

---

## L — **Liskov Substitution Principle (LSP)**  
*Subtypes must behave consistently with base types.*

### A-over-B reasoning:
✔ Choose **A (true subtype)** over B when subtype does NOT violate rules.

Violations example:
```java
class Square extends Rectangle { ... } // fails LSP often
```

---

## I — **Interface Segregation Principle (ISP)**  
*Prefer many small interfaces over large fat interfaces.*

### A-over-B reasoning:
✔ Prefer splitting interfaces (A)  
❌ Avoid "god interfaces" (B)

---

## D — **Dependency Inversion Principle (DIP)**  
*Depend on abstractions, not concretes.*

### A-over-B reasoning:
✔ Choose interfaces/ports (A)  
❌ Avoid direct dependencies on infrastructure (B)

---

# 1.2 DRY, DAMP, WET (REAL PRACTICAL)

## DRY — Don’t Repeat Yourself  
*Avoid duplicating knowledge.*

✔ Preferred when code is truly duplicating domain concepts.

## DAMP — Descriptive And Meaningful Phrases  
*Readable, expressive code > overly abstract code.*

Use DAMP instead of DRY when:
- removing duplication reduces clarity  
- abstraction forces indirection  

### Example:

### BAD DRY:
```java
public void process(Action a) {}
```

### GOOD DAMP:
```java
public void createOrder(CreateOrderCommand cmd)
public void payOrder(PayOrderCommand cmd)
```

---

## WET — Write Everything Twice  
*Sometimes duplication reduces coupling.*

✔ Good when systems are independent and shouldn’t share code.

Example:  
User domain in two microservices — duplication OK.

---

# 1.3 KISS — Keep It Simple, Stupid

✔ Choose simplest possible solution  
✔ Avoid premature abstraction  
✔ Avoid complicated inheritance  

---

# 1.4 YAGNI — You Aren’t Gonna Need It

Avoid designing features “for future flexibility” unless absolutely needed.

### A-over-B reasoning:
Choose **A (what is needed today)**  
Not **B (what might be needed later)**

---

# 1.5 GRASP Principles (Deep)

Key patterns:

- **Controller** → application layer  
- **Creator** → domain aggregates  
- **Low Coupling**  
- **High Cohesion**  
- **Information Expert**  
- **Polymorphism**  
- **Protected Variations**

---

# 1.6 Law of Demeter (LoD) — “Don’t talk to strangers”

BAD:
```java
order.getCustomer().getAddress().getCity();
```

GOOD:
```java
order.getCustomerCity();
```

---

# 1.7 Composition Over Inheritance (VERY IMPORTANT)

### A-over-B reasoning:

| Choose A (Composition) | Avoid B (Inheritance) |
|------------------------|-----------------------|
| when behavior varies | when inheritance is only for reuse |
| when classes change independently | when base class becomes god class |
| when future flexibility needed | when overrides break behavior |

Example:

### BAD:
```java
class EmailOrderService extends OrderService {}
```

### GOOD:
```java
class EmailNotifier { sendEmail() }
class OrderService {
    private EmailNotifier notifier;
}
```

---

# 2. DESIGN PATTERNS (Deep, Real Examples)

Patterns that matter the most in backend systems:

- Strategy (payments)  
- State (order workflow)  
- Builder (creating complex objects)  
- Chain of Responsibility (middleware)  
- Factory (resource creation)  
- Repository (DDD)  
- Observer / PubSub (domain events)  

Each pattern used based on **change expectation**, NOT personal preference.

---

# 3. DOMAIN-DRIVEN DESIGN (FULLY EXPLAINED EXAMPLES)

## 3.1 Entities & Value Objects

Example Value Object:

```java
record Money(BigDecimal amount, Currency currency) {}
```

Example Aggregate:

```java
class Order {
   List<OrderLine> lines;
   Status status;

   void addItem(...) { ... }
   void markPaid() { ... }
}
```

---

# 4. HEXAGONAL ARCHITECTURE (Ports + Adapters)

## Choosing A (Hexagonal) over B (Layered) when:
- integrating multiple external systems  
- need testability without infrastructure  
- long-term domain separation needed  

---

# 5. CLEAN ARCHITECTURE — Deep Examples

Choose A (Clean) over B (Traditional Layered) when:
- you want domain to be pure  
- business rules evolve frequently  
- you want infra to be replaceable  

---

# 6. CODE ANTI-PATTERNS (NOW WITH PRINCIPLES APPLIED)

## 6.1 God Class  
Violates SRP + Low Cohesion.

## 6.2 Anemic Domain  
Violates O (Open/Closed) and D (DIP).

## 6.3 Hard-coded Framework Logic  
Violates DIP.

---

# 7. REFACTORING USING PRINCIPLES (EXAMPLES)

### Example Problem:  
Service has too many responsibilities.

### Apply SRP:
Split into smaller cohesive services.

### Apply DIP:
Introduce interfaces.

### Apply DAMP:
Rename for clarity.

---

# 8. ARCHITECTURAL A-B DECISION MATRIX (VERY DEEP)

A vs B decisions:

| Decision | Choose A When | Choose B When |
|----------|---------------|---------------|
| Composition vs Inheritance | flexibility, decoupling | strict subtype relationship |
| DTO vs Entity | API boundaries | trivial monolith |
| Events vs Sync Calls | resilience, decoupling | immediate response needed |
| Hexagonal vs Layered | long-term complexity | small CRUD app |
| Split by Domain vs Split by Layer | team scaling | tiny project |

---

# 9. CODE REVIEW QUALITY GUIDELINES (DETAILED)

Reviewer checks:

- SRP followed?  
- DIP applied?  
- Not over-architected (YAGNI)?  
- Simplicity (KISS)?  
- No unnecessary abstractions (DAMP)?  
- Coupling minimized?  
- Cohesion high?  
- Patterns used properly?  

---

# END OF CHAPTER K
