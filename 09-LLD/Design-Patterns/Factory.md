# Factory Method Pattern

## Intent

Define an interface for creating an object, but let subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses, so the creating code depends on an abstraction rather than a concrete class.

---

## Problem

Client code often needs to create objects, but hardcoding `new ConcreteClassA()` directly into that code:

- Tightly couples the client to one specific concrete class — adding a new variant means finding and editing every place that constructs the old one
- Makes it hard to introduce new product types without touching existing, working client code (violates Open/Closed)
- Forces the client to know construction details (which constructor, which parameters, which concrete class fits which condition) that arguably belong somewhere else
- Becomes especially painful when the "right" class to instantiate depends on runtime conditions (config, input type, environment) — this logic ends up duplicated wherever objects are created

---

## Solution

Define a **Creator** with a `factoryMethod()` that returns a **Product** (usually via an interface or abstract base class). Subclasses of the Creator override `factoryMethod()` to instantiate a specific **ConcreteProduct**. The rest of the Creator's logic is written entirely against the Product interface, never against a concrete class — so it works unmodified no matter which ConcreteProduct a subclass decides to create.

- **Product (interface)** — declares the common interface for objects the factory method creates
- **ConcreteProduct** — a specific implementation of Product
- **Creator (often abstract)** — declares the `factoryMethod()`, and contains business logic that uses the Product it returns, without knowing which concrete type it got
- **ConcreteCreator** — overrides `factoryMethod()` to return a specific ConcreteProduct

A related, simpler variant sometimes conflated with this pattern in interviews is the **Simple Factory** (not a GoF pattern) — a single static method with a conditional that picks and returns the right ConcreteProduct based on input, with no subclassing involved. It solves the same "don't hardcode `new X()` everywhere" problem with less structure, at the cost of a conditional that must be edited for every new product type (see distinction in Interview Questions).

---

## Structure

```text
┌──────────────────────┐        ┌──────────────────┐
│   Creator (abstract)  │───────▶│ Product (interface)│
│ + factoryMethod():    │        │  + operation()       │
│     Product (abstract)│        └─────────┬─────────┘
│ + someOperation() {   │                  │ implemented by
│    p = factoryMethod()│                  ▼
│    p.operation()      │        ┌──────────────────┐
│  }                     │        │  ConcreteProductA │
└──────────┬────────────┘        └──────────────────┘
           │ extended by          ┌──────────────────┐
           ▼                      │  ConcreteProductB │
┌──────────────────────┐          └──────────────────┘
│  ConcreteCreatorA     │
│ + factoryMethod():     │
│     return ConcreteProductA()
└──────────────────────┘
┌──────────────────────┐
│  ConcreteCreatorB     │
│ + factoryMethod():     │
│     return ConcreteProductB()
└──────────────────────┘

Creator's someOperation() is written entirely against Product — never a concrete class.
Each ConcreteCreator decides, via factoryMethod(), which ConcreteProduct actually gets created.
```

---

## Example

```text
// Product interface
interface Notification {
    void send(String message);
}

// Concrete Products
class EmailNotification implements Notification {
    @Override
    public void send(String message) {
        System.out.println("Sending EMAIL: " + message);
    }
}

class SMSNotification implements Notification {
    @Override
    public void send(String message) {
        System.out.println("Sending SMS: " + message);
    }
}

// Creator (abstract) — business logic written only against Notification, never a concrete class
abstract class NotificationService {
    abstract Notification factoryMethod(); // subclasses decide what gets created

    void notifyUser(String message) {
        Notification notification = factoryMethod(); // Creator doesn't know which concrete type this is
        notification.send(message);
    }
}

// Concrete Creators — each overrides factoryMethod() to produce a specific Notification
class EmailNotificationService extends NotificationService {
    @Override
    Notification factoryMethod() {
        return new EmailNotification();
    }
}

class SMSNotificationService extends NotificationService {
    @Override
    Notification factoryMethod() {
        return new SMSNotification();
    }
}

// Client — depends only on the abstract NotificationService
NotificationService service = new EmailNotificationService();
service.notifyUser("Your order has shipped"); // Sending EMAIL: Your order has shipped

service = new SMSNotificationService();
service.notifyUser("Your OTP is 123456");     // Sending SMS: Your OTP is 123456
```

---

## When to Use

- A class can't anticipate the exact type of object it needs to create ahead of time — the decision should be deferred to subclasses or configuration
- You want to give users of a library/framework a way to **extend which product gets created**, without modifying the framework's own creation logic (a very common use in frameworks — e.g. a base class handles the workflow, subclasses plug in the specific object type)
- You want to eliminate tight coupling between client code and concrete classes, so new product types can be added by adding a new ConcreteCreator/ConcreteProduct pair, without touching existing code (Open/Closed)
- Object creation involves non-trivial logic (parsing config, choosing among several constructors, environment-based decisions) that shouldn't be duplicated at every call site

---

## When NOT to Use

- There's only one product type and no foreseeable need for variants — a plain constructor call is simpler and clearer
- The overhead of an abstract Creator + subclass hierarchy isn't justified by the actual variability — a **Simple Factory** (single method + conditional) is often sufficient and much less ceremony for a small, stable set of variants
- Introducing a new subclass hierarchy purely for object creation adds more complexity than the flexibility is worth for a small, rarely-changing codebase

---

## Trade-offs

|Pros|Cons|
|---|---|
|Decouples client/Creator code from concrete product classes|Requires creating a new Creator subclass for every new product variant, which can lead to parallel class hierarchies (a Creator subclass per Product subclass)|
|New product types can be added without modifying existing Creator code (Open/Closed)|Can be more complex than necessary for simple cases with few, stable variants — a Simple Factory or conditional may suffice|
|Centralizes and encapsulates object-creation logic, avoiding duplication across the codebase|Adds a layer of indirection that can make it harder to trace exactly which concrete class ends up being instantiated, especially in large hierarchies|
|Business logic in the Creator is written against an abstraction, making it easier to test/extend|Sometimes conflated with Abstract Factory or Simple Factory in interviews — precise naming matters (see Interview Questions)|

---

## Interview Questions

- **How is Factory Method different from Simple Factory?** Simple Factory isn't a GoF pattern — it's usually a single static method with a conditional/switch that picks and returns the right ConcreteProduct based on input. Factory Method uses **subclassing**: an abstract Creator declares the factory method, and each ConcreteCreator subclass overrides it to return a specific product — no conditional logic is needed because the _choice_ of subclass itself determines the product, and adding a new variant means adding a new subclass rather than editing a conditional.
- **How is Factory Method different from Abstract Factory?** Factory Method creates **one** product via a single overridden method. Abstract Factory creates **families of related products** (e.g. a `GUIFactory` that creates matching `Button`, `Checkbox`, and `Scrollbar` objects for a consistent look-and-feel) through multiple factory methods grouped in one interface — Abstract Factory is often implemented _using_ several Factory Methods internally.
- **How does Factory Method relate to the Open/Closed Principle?** Directly — the Creator's core logic (`someOperation()`) never needs to change when a new product type is introduced; you only add a new ConcreteCreator/ConcreteProduct pair, leaving existing, tested code untouched.
- **When would you prefer Simple Factory over Factory Method despite it not being a "real" GoF pattern?** When the set of product variants is small and relatively stable, and you don't need the extensibility of subclassing — a single factory method with a clear conditional is easier to read and navigate than a parallel hierarchy of Creator subclasses, each only existing to instantiate one Product type.
- **Can Factory Method be implemented without inheritance, e.g. in a language with first-class functions?** Yes — instead of subclassing a Creator, you can pass a factory **function/lambda** into the Creator's constructor or method, achieving the same decoupling (the Creator calls the injected function to get a Product) without a full class hierarchy — common in more functional-style codebases.
- **Where have you seen Factory Method used in real frameworks?** `java.util.Calendar.getInstance()` and `NumberFormat.getInstance()` (return different concrete implementations based on locale/config); GUI toolkit widget creation methods overridden per-platform; ORM frameworks that use factory methods to instantiate the correct entity subclass based on a discriminator column.

---

## Related Notes

- [[strategy]]
- [[singleton]]
- [[command]]