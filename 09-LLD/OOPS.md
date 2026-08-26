# Object-Oriented Programming

## Core Concepts

### Encapsulation

Bundling data (fields) and the methods that operate on it into a single unit (a class), while **restricting direct access** to internal state from outside — exposing a controlled interface instead. The goal is protecting an object's invariants: nothing outside the class can put it into an invalid state, because nothing outside the class can touch its internals directly.

```text
class BankAccount {
    private double balance;   // hidden — no direct external access

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Invalid amount");
        balance += amount;
    }

    public double getBalance() {
        return balance;       // controlled read access
    }
}
```

Without encapsulation (`public double balance`), any external code could set `balance = -500` directly, bypassing all validation. Encapsulation is what makes the class the single source of truth for its own correctness.

---

### Abstraction

Exposing only the **essential behavior** of an object while hiding the implementation details of how that behavior is achieved. Where encapsulation is about _hiding data_, abstraction is about _hiding complexity_ — the caller knows _what_ an object does, not _how_.

```text
interface PaymentGateway {
    void pay(double amount); // caller knows WHAT, not HOW
}

class StripeGateway implements PaymentGateway {
    public void pay(double amount) {
        // complex HTTP calls, retries, auth tokens — all hidden from the caller
    }
}
```

Achieved via **interfaces** and **abstract classes** — both let you define a contract without exposing (or even committing to) an implementation. This is the foundation for programming to an interface rather than a concrete class, which is what makes code testable and swappable.

---

### Inheritance

A class (**subclass**) acquires the fields and methods of another class (**superclass**), modeling an **is-a** relationship, and can override or extend behavior.

```text
class Animal {
    void eat() { System.out.println("eating"); }
}

class Dog extends Animal {
    void bark() { System.out.println("woof"); }
}
// Dog IS-A Animal — inherits eat(), adds bark()
```

- Enables **code reuse** — shared behavior lives once, in the superclass
- Enables **polymorphism** — a `Dog` can be used anywhere an `Animal` is expected
- **Risk:** deep inheritance hierarchies create tight coupling between subclass and superclass — a change to the superclass can ripple through every subclass, and subclasses are locked into a single parent (most OOP languages don't allow multiple class inheritance) — this is the core motivation behind "favor composition over inheritance" (see below)

---

### Polymorphism

The ability for objects of **different types** to be treated through a **common interface**, with each type providing its own specific behavior for the same call.

|Type|Mechanism|Example|
|---|---|---|
|**Runtime (subtype) polymorphism**|Method overriding — the actual method invoked is resolved at runtime based on the object's real type|`Animal a = new Dog(); a.makeSound();` calls `Dog`'s override, even though `a` is typed as `Animal`|
|**Compile-time polymorphism**|Method overloading — multiple methods with the same name but different parameter signatures, resolved at compile time|`add(int, int)` vs `add(double, double)` in the same class|

```text
class Animal {
    void makeSound() { System.out.println("..."); }
}
class Dog extends Animal {
    @Override void makeSound() { System.out.println("Woof"); }
}
class Cat extends Animal {
    @Override void makeSound() { System.out.println("Meow"); }
}

List<Animal> animals = List.of(new Dog(), new Cat());
for (Animal a : animals) a.makeSound(); // "Woof", then "Meow" — same call, different behavior
```

Runtime polymorphism is what makes [[strategy]], [[decorator]], and most other behavioral design patterns possible — a client can call the same method on any object implementing a shared interface, without knowing (or caring) which concrete class it actually is.

---

### Composition

Building complex objects by **combining other objects** as fields, modeling a **has-a** relationship, rather than inheriting behavior from a parent class.

```text
class Engine {
    void start() { System.out.println("Engine starting"); }
}

class Car {
    private final Engine engine; // Car HAS-A Engine, not IS-A Engine

    Car(Engine engine) { this.engine = engine; }

    void start() { engine.start(); } // delegate
}
```

**"Favor composition over inheritance"** — a foundational OOP design principle:

| |Inheritance|Composition|
|---|---|---|
|Relationship|is-a|has-a|
|Coupling|Tight — subclass depends on superclass internals, breaks easily on superclass changes|Loose — depends only on the composed object's interface|
|Flexibility|Fixed at compile time, single parent (in most languages)|Can be swapped/reconfigured at runtime (e.g. inject a different `Engine`)|
|Reuse across unrelated hierarchies|Hard — behavior is locked to the class hierarchy|Easy — any class can hold a reference to any other and delegate to it|

Composition is the structural basis of most GoF design patterns — [[strategy]] swaps composed behavior at runtime, [[decorator]] wraps a composed component, [[adapter]] wraps a composed adaptee. When a design question asks "would you use inheritance or composition here," composition is very often the safer default unless there's a genuine, stable is-a relationship.

---

## Design Questions

- **Encapsulation vs Abstraction — what's the actual difference?** They're often confused because both involve "hiding" something. Encapsulation hides **data** (implementation state) via access modifiers — it's about _protecting_ internals. Abstraction hides **complexity** (implementation logic) via interfaces/abstract classes — it's about _simplifying_ what the caller needs to know. A class can be encapsulated without being abstract (private fields, but a concrete class with no interface), and abstraction can exist without much encapsulation (an interface says nothing about how implementers store their data).
    
- **Why favor composition over inheritance?** Inheritance creates the tightest possible coupling between two classes — a subclass's correctness depends on the superclass's internal implementation details, not just its public contract (the "fragile base class" problem: a seemingly safe change to the superclass can silently break subclasses). Composition only depends on the composed object's public interface, can be reconfigured at runtime (swap in a different implementation via dependency injection), and avoids being locked into one fixed hierarchy. Inheritance is still the right tool for a genuine, stable is-a relationship (e.g. `Circle`/`Square` extending `Shape`); composition wins almost everywhere else.
    
- **What is the Liskov Substitution Principle, and how does it relate to inheritance?** A subclass should be substitutable for its superclass without breaking the correctness of the program — i.e. any code that works with the superclass should keep working if you swap in a subclass instance. The classic violation example: `Square extends Rectangle` — if `Rectangle` has independent `setWidth()`/`setHeight()`, a `Square` overriding both to stay square breaks any code that assumes setting width alone doesn't affect height. This is a strong signal that the is-a relationship was modeled incorrectly, and composition (or a shared interface without the subtype relationship) is often the better fix.
    
- **Can you have polymorphism without inheritance?** Yes — via interfaces. A class doesn't need to inherit from a common superclass to be polymorphic; implementing a shared interface (`Comparable`, `Runnable`, a custom `PaymentGateway`) is enough for runtime polymorphism to work, since the calling code only cares about the interface, not the concrete class or its ancestry. This is why "program to an interface, not an implementation" is often stated as a design principle independent of inheritance itself.
    
- **What's the difference between an abstract class and an interface, and when would you choose one over the other?** An abstract class can hold **state** (fields) and **partial implementation** (some concrete methods, some abstract) and supports only single inheritance. An interface (in most languages, pre default-methods) is a pure contract with no state, and a class can implement **multiple** interfaces. Choose an abstract class when subclasses share meaningful common state/behavior and a genuine is-a hierarchy exists; choose an interface when you only need to guarantee a contract across otherwise unrelated classes (an interface answers "can you do X," not "what are you").
    
- **How does encapsulation support the Open/Closed Principle?** By hiding internal state and exposing only a stable public interface, a class's internals can be freely changed (optimized, refactored, re-implemented) without affecting any code that depends on it — as long as the public contract doesn't change. This is what lets a codebase be "closed for modification, open for extension": callers extend behavior by composing with or subclassing the public interface, never by reaching into internals that were never meant to be touched.
    
- **Give a real design scenario where you'd choose composition over inheritance.** A `Car` needing an `Engine`: modeling `Car extends Engine` would be wrong (a Car is not a kind of Engine — that's not an is-a relationship at all), so composition (`Car has-a Engine`) is the only sensible model. Beyond just correctness, composition here also lets you swap engine types at runtime (`ElectricEngine` vs `GasEngine`) by injecting a different `Engine` implementation, without touching `Car`'s class definition — exactly the flexibility a rigid inheritance hierarchy would prevent.
    

---

## Related Notes

- [[strategy]]
- [[decorator]]
- [[adapter]]
- [[builder]]