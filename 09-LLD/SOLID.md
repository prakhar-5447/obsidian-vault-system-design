# SOLID

Five design principles for writing maintainable, extensible object-oriented code — coined/popularized by Robert C. Martin ("Uncle Bob"). They're guidelines for **where to draw class/module boundaries** so a codebase stays easy to change as requirements evolve, not rigid syntax rules.

---

## SRP — Single Responsibility Principle

**A class should have only one reason to change.**

Not "a class should do only one thing" (too vague to apply) — the precise formulation is about **ownership**: each responsibility should be owned by exactly one actor/stakeholder, so a change requested by one stakeholder never forces a change to code another stakeholder depends on.

**Violation:**

```text
class Invoice {
    void calculateTotal() { /* business logic */ }
    void printInvoice() { /* formatting/presentation logic */ }
    void saveToDatabase() { /* persistence logic */ }
}
// Three reasons to change: pricing rules change, print format changes,
// or the database schema/technology changes — each touches this ONE class.
```

**Fix — split by responsibility/reason to change:**

```text
class Invoice {
    void calculateTotal() { /* business logic only */ }
}

class InvoicePrinter {
    void print(Invoice invoice) { /* formatting only */ }
}

class InvoiceRepository {
    void save(Invoice invoice) { /* persistence only */ }
}
```

**Why it matters:** a class with multiple responsibilities has multiple, unrelated stakeholders touching the same code — changes for one reason risk breaking behavior that has nothing to do with that change, and the class becomes a merge-conflict/regression magnet.

---

## OCP — Open/Closed Principle

**Software entities should be open for extension, but closed for modification.**

You should be able to add new behavior **without changing existing, already-tested code** — new functionality comes from adding new code (new classes implementing an existing interface), not editing what's already there.

**Violation:**

```text
class DiscountCalculator {
    double calculate(String customerType, double amount) {
        if (customerType.equals("REGULAR")) return amount * 0.95;
        if (customerType.equals("PREMIUM")) return amount * 0.90;
        if (customerType.equals("VIP")) return amount * 0.80;
        // every new customer type means editing this method again
        return amount;
    }
}
```

**Fix — polymorphism instead of conditionals ([[strategy]] pattern):**

```text
interface DiscountStrategy {
    double apply(double amount);
}
class RegularDiscount implements DiscountStrategy {
    public double apply(double amount) { return amount * 0.95; }
}
class PremiumDiscount implements DiscountStrategy {
    public double apply(double amount) { return amount * 0.90; }
}
// Adding VIP later = add ONE new class. Zero existing code touched.
class VipDiscount implements DiscountStrategy {
    public double apply(double amount) { return amount * 0.80; }
}
```

**Why it matters:** every time you modify existing, working, tested code, you risk breaking it. OCP-compliant designs let new requirements be satisfied purely additively — this is the principle behind almost every "pluggable strategy/handler" design in this vault ([[strategy]], [[chain-of-responsibility]], [[adapter]]).

---

## LSP — Liskov Substitution Principle

**Subtypes must be substitutable for their base types without altering the correctness of the program.**

If `S` is a subtype of `T`, any code written to work with `T` should keep working correctly if you hand it an `S` instead — a subclass shouldn't strengthen preconditions, weaken postconditions, or violate any behavioral contract the base type promised.

**Violation — the classic Square/Rectangle problem:**

```text
class Rectangle {
    protected int width, height;
    void setWidth(int w) { width = w; }
    void setHeight(int h) { height = h; }
    int area() { return width * height; }
}

class Square extends Rectangle {
    // A square must keep width == height, so it overrides both setters:
    @Override void setWidth(int w) { width = height = w; }
    @Override void setHeight(int h) { width = height = h; }
}

// Code written against Rectangle breaks silently when given a Square:
void resize(Rectangle r) {
    r.setWidth(5);
    r.setHeight(10);
    assert r.area() == 50; // TRUE for Rectangle, FALSE for Square (area = 100)
}
```

`Square` is a valid _mathematical_ is-a Rectangle, but not a valid _behavioral_ one — it breaks an assumption (`setWidth` doesn't affect height) that calling code is entitled to rely on.

**Fix — don't force the is-a relationship where behavior actually differs:**

```text
interface Shape {
    int area();
}
class Rectangle implements Shape { /* independent width/height */ }
class Square implements Shape { /* single side field, no setWidth/setHeight at all */ }
// No inheritance between them — just a shared Shape contract for what both CAN do.
```

**Why it matters:** LSP violations are usually a sign that an is-a relationship was modeled where a has-a or a shared-interface-without-inheritance relationship should have been used instead — directly connects to the [[oops|composition over inheritance]] discussion.

---

## ISP — Interface Segregation Principle

**No client should be forced to depend on methods it does not use.**

Prefer several small, focused interfaces over one large, general-purpose interface — a "fat" interface forces every implementer to provide (or stub out/throw on) methods that are irrelevant to it.

**Violation:**

```text
interface Worker {
    void work();
    void eat();
}

class HumanWorker implements Worker {
    public void work() { /* ... */ }
    public void eat() { /* ... */ }   // makes sense
}

class RobotWorker implements Worker {
    public void work() { /* ... */ }
    public void eat() { throw new UnsupportedOperationException(); } // forced, meaningless
}
```

**Fix — split into focused interfaces:**

```text
interface Workable {
    void work();
}
interface Eatable {
    void eat();
}

class HumanWorker implements Workable, Eatable {
    public void work() { /* ... */ }
    public void eat() { /* ... */ }
}
class RobotWorker implements Workable {
    public void work() { /* ... */ } // no eat() to fake — RobotWorker was never Eatable
}
```

**Why it matters:** fat interfaces force implementers into fake/throwing method bodies (a red flag), and force _callers_ of the interface to be coupled to methods they'll never call — any change to an unrelated method in the fat interface can still ripple to every implementer, even the ones that don't care about it.

---

## DIP — Dependency Inversion Principle

**High-level modules should not depend on low-level modules; both should depend on abstractions. Abstractions should not depend on details; details should depend on abstractions.**

Business/policy logic (high-level) shouldn't be wired directly to concrete implementation details (low-level, e.g. a specific database driver or HTTP client) — both should depend on an interface sitting between them, so the low-level detail can be swapped without touching the high-level logic.

**Violation:**

```text
class MySQLDatabase {
    void save(String data) { /* MySQL-specific code */ }
}

class OrderService {
    private MySQLDatabase db = new MySQLDatabase(); // hard dependency on a CONCRETE class
    void placeOrder(String order) { db.save(order); }
}
// Switching to PostgreSQL, or unit-testing OrderService without a real DB, means editing OrderService.
```

**Fix — depend on an abstraction, inject the concrete implementation:**

```text
interface Database {
    void save(String data);
}
class MySQLDatabase implements Database {
    public void save(String data) { /* MySQL-specific code */ }
}
class PostgresDatabase implements Database {
    public void save(String data) { /* Postgres-specific code */ }
}

class OrderService {
    private final Database db; // depends on the ABSTRACTION
    OrderService(Database db) { this.db = db; } // injected — dependency inversion
    void placeOrder(String order) { db.save(order); }
}
// OrderService.java never changes when swapping databases, and is trivially
// unit-testable by injecting a fake/mock Database implementation.
```

**Why it matters:** this is the principle behind **dependency injection** as a practice — "inversion" refers to the fact that traditionally the high-level module would own/construct its low-level dependency directly (control flows top-down); DIP inverts that so the dependency is handed in from outside, and both sides point at a shared abstraction instead of the high-level module pointing down at the concrete low-level one.

---

## Examples

Quick reference tying each violation/fix pattern to the principle it demonstrates:

|Principle|Symptom of violation|Typical fix|
|---|---|---|
|**SRP**|A class changes for unrelated reasons (business logic + formatting + persistence all in one class)|Split into separate classes, one per responsibility/stakeholder|
|**OCP**|Adding a new case means editing an existing `if/else` or `switch`|Extract a polymorphic interface; new cases = new classes ([[strategy]])|
|**LSP**|A subclass overrides a method in a way that breaks an assumption callers relied on|Don't force an is-a relationship where behavior genuinely diverges; use a shared interface without inheritance, or composition|
|**ISP**|An implementer has to stub out or throw on a method it doesn't actually support|Split the fat interface into several small, role-specific interfaces|
|**DIP**|A high-level class directly instantiates (`new`s) a concrete low-level class|Depend on an interface; inject the concrete implementation from outside|

**How the five reinforce each other in one design:** SRP keeps each class focused enough that OCP-style extension (adding new classes) is clean; DIP's abstraction layer is what OCP's "new implementation, zero existing code touched" is built on top of; ISP keeps those abstractions small enough that LSP substitutability is easy to actually guarantee (a fat interface is much harder to implement correctly and substitutably in every case).

**How this maps onto design patterns already in this vault:**

- [[strategy]] and [[chain-of-responsibility]] are direct applications of **OCP** (swap/extend behavior without modifying existing code)
- [[adapter]] is a direct application of **DIP** (client depends on the Target interface, not the concrete Adaptee)
- [[decorator]] relies on **LSP** — a decorated component must remain substitutable for an undecorated one, since both implement the same `Component` interface
- [[builder]]'s immutable Product and [[command]]'s Receiver/Invoker separation both lean on **SRP** (construction logic vs the object itself; triggering an action vs performing it)

---

## Related Notes

- [[oops]]
- [[strategy]]
- [[adapter]]
- [[decorator]]
- [[command]]
- [[chain-of-responsibility]]