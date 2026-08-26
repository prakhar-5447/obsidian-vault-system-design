# Decorator Pattern

## Intent

Attach additional responsibilities to an object dynamically. Decorator provides a flexible alternative to subclassing for extending behavior, by wrapping an object with one or more Decorators that add behavior while conforming to the same interface as the object they wrap.

---

## Problem

You want to add extra behavior/responsibilities to individual objects (e.g. adding scrollbars or borders to a UI component, adding compression or encryption to a data stream, adding extra toppings to a coffee order) — but:

- Using subclassing to cover every possible combination of extra behaviors leads to a **combinatorial explosion** of subclasses (e.g. `CompressedEncryptedStream`, `EncryptedCompressedStream`, `CompressedStream`, `EncryptedStream`, and so on for every combination)
- Some behaviors should be addable/removable **per object instance**, at runtime — not baked into the class hierarchy at compile time for every object of that class
- Subclassing extends behavior for an entire class of objects; you often only want to extend behavior for **one specific instance**, leaving other instances of the same class unaffected

---

## Solution

Create a **Decorator** class that implements the **same interface** as the object it wraps (the **Component**), holds a reference to a wrapped Component, and delegates the core operation to it — while adding its own behavior before or after that delegation. Because Decorators implement the same interface as the Component, they can be **stacked** — a Decorator can wrap another Decorator, which wraps another, and so on, each layer adding its own behavior.

The client interacts with the outermost wrapper exactly as it would with the original Component — it doesn't need to know how many layers of decoration are involved, or in what order.

- **Component (interface)** — the common interface for both the original object and all Decorators
- **ConcreteComponent** — the original, undecorated object
- **Decorator (abstract, often)** — implements Component, holds a reference to a wrapped Component, delegates the core call to it
- **ConcreteDecorator** — adds specific extra behavior before/after delegating to the wrapped Component

---

## Structure

```text
┌────────────────────┐
│  Component (interface)│
│  + operation()         │
└──────────┬─────────┘
           │ implemented by
   ┌───────┴────────┐
   ▼                 ▼
┌──────────────┐  ┌─────────────────────┐
│ConcreteComponent│ │  Decorator (abstract)│
│ + operation()   │ │ - wrappee: Component  │
└──────────────┘  │ + operation() {         │
                    │    wrappee.operation()  │
                    │  }                        │
                    └──────────┬───────────┘
                               │ extended by
                    ┌──────────┴───────────┐
                    ▼                       ▼
        ┌──────────────────┐   ┌──────────────────┐
        │ ConcreteDecoratorA │   │ ConcreteDecoratorB │
        │ + operation() {      │   │ + operation() {      │
        │    super.operation() │   │    // extra behavior   │
        │    // extra behavior │   │    super.operation()   │
        │  }                     │   │  }                     │
        └──────────────────┘   └──────────────────┘

Decorators can be nested arbitrarily: new DecoratorB(new DecoratorA(new ConcreteComponent()))
Each layer wraps the previous, adding its own behavior while still honoring the Component interface.
```

---

## Example

```text
// Component interface
interface Coffee {
    double cost();
    String description();
}

// ConcreteComponent — the base object being decorated
class SimpleCoffee implements Coffee {
    @Override
    public double cost() { return 2.00; }

    @Override
    public String description() { return "Coffee"; }
}

// Decorator (abstract) — implements Component, wraps another Component
abstract class CoffeeDecorator implements Coffee {
    protected final Coffee wrappee;

    CoffeeDecorator(Coffee wrappee) {
        this.wrappee = wrappee;
    }
}

// Concrete Decorators — each adds its own cost/description, then delegates
class MilkDecorator extends CoffeeDecorator {
    MilkDecorator(Coffee wrappee) { super(wrappee); }

    @Override
    public double cost() { return wrappee.cost() + 0.50; }

    @Override
    public String description() { return wrappee.description() + " + Milk"; }
}

class WhippedCreamDecorator extends CoffeeDecorator {
    WhippedCreamDecorator(Coffee wrappee) { super(wrappee); }

    @Override
    public double cost() { return wrappee.cost() + 0.70; }

    @Override
    public String description() { return wrappee.description() + " + Whipped Cream"; }
}

// Client — stacks decorators to build up behavior, one instance at a time
Coffee order = new SimpleCoffee();
System.out.println(order.description() + " = $" + order.cost());
// Coffee = $2.0

order = new MilkDecorator(order);
order = new WhippedCreamDecorator(order);
System.out.println(order.description() + " = $" + order.cost());
// Coffee + Milk + Whipped Cream = $3.2

// A different SimpleCoffee instance remains completely unaffected — decoration is per-instance
Coffee plainOrder = new SimpleCoffee();
System.out.println(plainOrder.description() + " = $" + plainOrder.cost());
// Coffee = $2.0
```

---

## When to Use

- You need to add responsibilities to **individual objects** at runtime, without affecting other instances of the same class
- You want to avoid a combinatorial explosion of subclasses for every possible combination of optional behaviors — Decorator lets you compose behaviors instead of hardcoding every combination as its own class
- You want extensions that can be added and removed dynamically, or stacked in different orders to produce different combined effects
- The core object and its extensions should conform to the same interface, so decorated and undecorated objects are interchangeable from the client's point of view

---

## When NOT to Use

- The set of possible extensions is small, fixed, and unlikely to grow or combine — plain subclassing (or even just optional constructor parameters/flags) may be simpler and easier to follow
- Deeply nested decorator chains become hard to read/debug — tracing behavior through many stacked layers can obscure what's actually happening, especially if the wrapping order matters and isn't obvious from the code
- The wrapped behaviors need to interact with or depend on each other's internal state in complex ways — Decorators are meant to be relatively independent, composable layers; tightly interdependent extensions may fit a different pattern better

---

## Trade-offs

|Pros|Cons|
|---|---|
|Adds behavior to individual objects at runtime without affecting other instances|Can result in many small Decorator classes, and a system with a large number of small objects that can be hard to understand at a glance|
|Avoids a combinatorial subclass explosion for every combination of optional features|Order of wrapping can matter and produce different results — this can be a subtle source of bugs if not carefully documented/tested|
|Decorators can be combined/stacked flexibly, supporting many combinations from a small set of building blocks|Removing a specific Decorator from the middle of an existing stack (not just the outermost layer) is awkward — decorators are naturally suited to adding/removing from the outside only|
|Follows Open/Closed — new behavior can be added as new Decorator classes without modifying existing Component or Decorator code|Debugging can be harder because the "real" object is buried inside potentially several layers of wrapping, and stack traces get deeper|

---

## Interview Questions

- **How is Decorator different from Proxy?** Both wrap another object behind the same interface, and are structurally almost identical. The difference is intent: [[proxy|Proxy]] controls **access** to the wrapped object (deciding whether/when/how the call reaches it — permission checks, lazy loading, remoting) and typically wraps exactly **one** RealSubject. Decorator **adds new behavior/responsibilities** on top of an object it always fully delegates to, and is explicitly designed to be **stacked** in multiple layers, which Proxy usually isn't.
- **How is Decorator different from subclassing/inheritance?** Inheritance extends behavior for an entire class **at compile time** — every instance of the subclass gets the new behavior. Decorator extends behavior for **individual object instances at runtime** — you choose which specific objects get wrapped, and can combine multiple decorators in different orders per instance, which a fixed class hierarchy can't express without a subclass for every combination.
- **Does the order in which you apply Decorators matter?** Often yes — e.g. "compress then encrypt" produces different bytes than "encrypt then compress" (compressing already-encrypted, high-entropy data barely shrinks it at all) — this is a common, concrete interview example of why decoration order is a real design decision, not an implementation detail to ignore.
- **How would you remove one specific Decorator from the middle of a stack?** This is awkward by design — Decorators only naturally support adding/removing from the _outermost_ layer. To truly "remove" one from the middle, you'd typically need to rebuild the stack without that layer (unwrap down to the target, then rewrap the remaining layers around it), which is why Decorator is best suited to add-only or add/remove-from-outside use cases.
- **What's a well-known real-world example of Decorator?** Java's I/O stream classes (`BufferedInputStream(new FileInputStream(...))`, optionally further wrapped in `GZIPInputStream` or `ObjectInputStream`) are a textbook Decorator implementation — each wrapper adds buffering, compression, or object serialization on top of the same `InputStream` interface, and can be composed in different combinations.
- **Is Decorator related to Composite?** They're often used together — a Decorator can be thought of as a [[composite|Composite]] with exactly one child, always delegating to it, plus its own extra behavior. Both rely on treating wrapped/child objects through a common interface, but Composite's intent is representing part-whole tree hierarchies uniformly, while Decorator's intent is layering additional responsibilities onto a single object.

---

## Related Notes

- [[proxy]]
- [[facade]]
- [[command]]