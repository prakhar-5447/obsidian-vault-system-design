# Strategy Pattern

## Intent

Define a family of algorithms, encapsulate each one as an object, and make them interchangeable. Strategy lets the algorithm vary independently from the clients that use it.

---

## Problem

You have a task that can be accomplished in **multiple ways** (e.g. sorting, compression, payment processing, route calculation), and:

- You don't want to hardcode one specific algorithm into the client, because the right choice may depend on context, configuration, or runtime conditions
- You don't want a giant `if/else` or `switch` block picking between algorithm variants scattered through the codebase — this violates Open/Closed (adding a new variant means editing existing code) and duplicates the same conditional logic wherever the algorithm is used
- You want to be able to **swap the algorithm at runtime**, or configure it per-instance, without changing the client's code

---

## Solution

Extract each algorithm variant into its own class implementing a common **Strategy** interface (typically a single method, e.g. `execute()` or `apply()`). The **Context** class holds a reference to a Strategy object and delegates the actual work to it, instead of implementing the algorithm itself.

The client chooses which concrete Strategy to hand to the Context — the Context itself never knows or cares which variant it's running, only that it conforms to the Strategy interface.

- **Context** — the object that needs the algorithm, holds a Strategy reference, delegates to it
- **Strategy (interface)** — declares the common operation all algorithm variants must implement
- **ConcreteStrategy** — one specific algorithm implementation

---

## Structure

```text
┌────────────┐        ┌──────────────────┐
│  Context   │───────▶│ Strategy (interface)│
│ - strategy │        │  + execute()         │
│ + setStrategy()│     └─────────┬─────────┘
│ + doWork() {   │               │ implemented by
│    strategy.execute()│         ▼
│  }             │      ┌──────────────────┐
└────────────┘         │ ConcreteStrategyA │
                        │  + execute()       │
                        └──────────────────┘
                        ┌──────────────────┐
                        │ ConcreteStrategyB │
                        │  + execute()       │
                        └──────────────────┘

Client selects a ConcreteStrategy and injects it into the Context.
Context delegates all algorithm-specific work to whichever Strategy it currently holds.
```

---

## Example

```text
// Strategy interface
interface PaymentStrategy {
    void pay(int amount);
}

// Concrete Strategies — each is a swappable algorithm
class CreditCardPayment implements PaymentStrategy {
    private final String cardNumber;

    CreditCardPayment(String cardNumber) {
        this.cardNumber = cardNumber;
    }

    @Override
    public void pay(int amount) {
        System.out.println("Paid " + amount + " using Credit Card " + cardNumber);
    }
}

class PayPalPayment implements PaymentStrategy {
    private final String email;

    PayPalPayment(String email) {
        this.email = email;
    }

    @Override
    public void pay(int amount) {
        System.out.println("Paid " + amount + " using PayPal account " + email);
    }
}

// Context — delegates to whichever strategy it holds, doesn't know the concrete type
class ShoppingCart {
    private PaymentStrategy paymentStrategy;

    void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }

    void checkout(int amount) {
        paymentStrategy.pay(amount); // delegates — doesn't know HOW payment happens
    }
}

// Client — chooses and injects the strategy
ShoppingCart cart = new ShoppingCart();
cart.setPaymentStrategy(new CreditCardPayment("1234-5678"));
cart.checkout(500);                                    // Paid 500 using Credit Card 1234-5678

cart.setPaymentStrategy(new PayPalPayment("user@mail.com"));
cart.checkout(200);                                    // Paid 200 using PayPal account user@mail.com
```

---

## When to Use

- You have **multiple variants** of an algorithm and need to switch between them, possibly at runtime, without changing the client that uses them
- You want to eliminate large conditional blocks (`if/else`, `switch`) that select between related behaviors — Strategy replaces the conditional with polymorphism
- You want to isolate an algorithm's implementation details from the code that uses it, so each variant can be tested, changed, or extended independently
- Different clients/contexts need different behavior for the same conceptual operation (e.g. different sorting comparators, different compression algorithms, different pricing rules)

---

## When NOT to Use

- There's genuinely only **one** algorithm and no foreseeable need for variants — introducing a Strategy interface and Context indirection is unnecessary ceremony
- The algorithms are trivially simple (a one-line difference) — a simple parameter or lambda/function reference may be enough without a full class hierarchy
- The number of strategy variants is small and stable, and a simple conditional is actually clearer to readers than several small classes

---

## Trade-offs

|Pros|Cons|
|---|---|
|Eliminates conditional logic for selecting between algorithm variants|Client must be aware of the different strategies to choose the right one|
|New strategies can be added without modifying the Context (Open/Closed)|Increases the number of classes/objects in the system|
|Algorithms can be swapped at runtime, even per-instance|Communication overhead between Strategy and Context if the algorithm needs a lot of context data (may require passing extra parameters or context references)|
|Each algorithm is isolated, easier to test independently|For very simple variations, the pattern can feel like overkill compared to a function pointer/lambda|

---

## Interview Questions

- **How is Strategy different from Command?** Strategy is about choosing **which algorithm** to run for a given operation — the interchangeable implementations all conceptually do "the same job" differently (e.g. different sorting algorithms). Command is about representing **a request** as an object, decoupling invoker from receiver, and typically supporting queuing, logging, and undo — concerns Strategy doesn't address.
- **How is Strategy different from State?** Both use composition and delegate to an interchangeable object, and the structure diagrams look almost identical. The difference is intent and awareness: in Strategy, the **client** chooses the algorithm and the strategies are usually unaware of each other. In [[state]], the **Context** transitions between states based on its own logic, and states are often aware of, and trigger transitions to, other states. Strategy variants are typically independent; State variants typically model a related, transitioning set of behaviors for one object's lifecycle.
- **Can Strategy be implemented without a class hierarchy?** In languages with first-class functions (Python, JavaScript, Java with lambdas), a simple function/lambda can serve as a lightweight Strategy — you don't always need a full interface + concrete classes if the "algorithm" is just a single method with no extra state.
- **How would you let the Context choose a default strategy without the client specifying one?** Give the Context a sensible default Strategy in its constructor, with a setter allowing the client to override it later — avoids forcing every client to always specify a strategy explicitly.
- **Where have you seen Strategy used in real systems/frameworks?** `Comparator` implementations passed to `Collections.sort()`; different compression codecs (gzip, brotli) selected behind a common interface; payment gateway integrations (Stripe, PayPal, etc.) behind a common `PaymentStrategy`; validation rule engines where different validation strategies apply to different field types.
- **Does Strategy violate encapsulation by requiring the client to know about concrete strategy classes?** To some degree, yes — the client must know enough about the available strategies to choose the right one. This is often mitigated with a **Factory** that maps a simpler input (e.g. a config string) to the correct concrete Strategy, so the client doesn't need to import/instantiate concrete Strategy classes directly.

---

## Related Notes

- [[state]]
- [[command]]
- [[factory-method]]
- [[observer]]