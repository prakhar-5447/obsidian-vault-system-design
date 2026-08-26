# Adapter Pattern

## Intent

Convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces — without modifying either the client or the existing (often third-party or legacy) class.

---

## Problem

You have an existing class (or third-party library) whose interface **doesn't match** what your client code expects, and you can't (or shouldn't) change that existing class:

- It's from a **third-party library** — you don't own the source
- It's **legacy code** — modifying it risks breaking other consumers
- You want to **swap implementations** later (e.g. switch payment providers) without rewriting every call site

Directly wiring the client to the incompatible class either doesn't compile/work, or forces you to scatter conversion logic across every call site instead of in one place.

---

## Solution

Introduce an **Adapter** class that sits between the client and the incompatible class (the **Adaptee**). The Adapter implements the interface the client expects (the **Target**), and internally translates/delegates calls to the Adaptee's actual interface.

- **Object Adapter** (composition) — Adapter holds a reference to an Adaptee instance and delegates to it. Preferred in most cases — works even when you can't subclass the Adaptee (e.g. it's `final`/sealed), and can adapt any subclass of the Adaptee at runtime.
- **Class Adapter** (inheritance) — Adapter extends the Adaptee and implements the Target interface simultaneously. Only possible in languages with multiple inheritance (or via interfaces + a single concrete base in Java/C#); less flexible than the object adapter.

---

## Structure

```text
┌─────────────┐        ┌───────────────────┐        ┌────────────────┐
│   Client    │───────▶│   Target (interface)│◀──────│    Adapter     │
└─────────────┘        └───────────────────┘        └────────┬───────┘
                                                               │ delegates to
                                                               ▼
                                                      ┌────────────────┐
                                                      │    Adaptee     │
                                                      │ (incompatible  │
                                                      │  interface)    │
                                                      └────────────────┘

Client depends only on Target.
Adapter implements Target and wraps/holds an Adaptee instance.
Client never talks to Adaptee directly.
```

---

## Example

```text
// Target interface — what the client code expects to call
interface PaymentProcessor {
    void pay(double amountInDollars);
}

// Adaptee — existing third-party library, incompatible interface,
// takes cents as an integer and uses a different method name
class LegacyStripeGateway {
    void makeChargeInCents(int amountInCents) {
        System.out.println("Charging " + amountInCents + " cents via Stripe");
    }
}

// Adapter — implements Target, translates calls to Adaptee
class StripeAdapter implements PaymentProcessor {
    private final LegacyStripeGateway stripeGateway;

    StripeAdapter(LegacyStripeGateway stripeGateway) {
        this.stripeGateway = stripeGateway;
    }

    @Override
    public void pay(double amountInDollars) {
        int cents = (int) Math.round(amountInDollars * 100);
        stripeGateway.makeChargeInCents(cents);
    }
}

// Client code — only ever depends on PaymentProcessor (Target)
class CheckoutService {
    private final PaymentProcessor paymentProcessor;

    CheckoutService(PaymentProcessor paymentProcessor) {
        this.paymentProcessor = paymentProcessor;
    }

    void checkout(double total) {
        paymentProcessor.pay(total);
    }
}

// Wiring
PaymentProcessor processor = new StripeAdapter(new LegacyStripeGateway());
CheckoutService checkout = new CheckoutService(processor);
checkout.checkout(49.99); // client is unaware LegacyStripeGateway exists
```

---

## When to Use

- Integrating a **third-party library or external API** whose interface doesn't match your application's expected interface
- Working with **legacy code** you can't modify but need to fit into a newer interface/abstraction
- You want to let two independently-designed classes/systems collaborate without altering either one's source
- You anticipate **swapping implementations** later (multiple payment gateways, multiple storage backends) and want a single, consistent interface for the rest of the app to depend on

---

## When NOT to Use

- You **own and can freely modify** the incompatible class — just change its interface directly instead of adding a wrapping layer
- The interfaces are already compatible or only trivially different — an adapter adds indirection with no real benefit
- You need to change the **behavior**, not just the interface shape — that's [[decorator]], not Adapter (Adapter only translates the interface, it doesn't add new responsibilities)
- Overusing Adapter across many unrelated classes can turn into a maze of thin wrapper classes that obscure what's actually happening — sometimes a small amount of duplicated conversion code is more readable than a fleet of adapters

---

## Trade-offs

|Pros|Cons|
|---|---|
|Lets incompatible interfaces work together without modifying existing code|Adds an extra layer of indirection — one more class to read through|
|Single Responsibility — conversion logic lives in one place instead of scattered at call sites|Class Adapter (inheritance-based) is limited by single-inheritance constraints in many languages|
|Open/Closed — introduce new adapters without touching existing client or adaptee code|If overused, can result in many thin wrapper classes that make the codebase harder to navigate|
|Makes swapping implementations easy (client only depends on the Target interface)|Doesn't help if the mismatch is behavioral, not just structural (interface shape)|

---

## Interview Questions

- **How is Adapter different from Decorator?** Adapter changes an interface to match what the client expects, without adding new behavior. Decorator keeps the same interface but adds new responsibilities/behavior dynamically.
- **How is Adapter different from Facade?** Adapter makes one existing interface look like another the client already expects (interface translation, usually 1:1). Facade creates a **new, simplified** interface over a complex subsystem, often wrapping many classes at once — it's about simplification, not compatibility.
- **Object Adapter vs Class Adapter — which do you prefer and why?** Object Adapter (composition) is generally preferred — it can adapt any subclass of the Adaptee at runtime, doesn't require multiple inheritance, and follows "favor composition over inheritance." Class Adapter is more rigid but can override Adaptee behavior directly since it _is_ an Adaptee.
- **Where have you seen Adapter used in real libraries?** Java's `Arrays.asList()` adapting an array to the `List` interface; `InputStreamReader` adapting a byte stream (`InputStream`) to a character stream (`Reader`); Spring's `HandlerAdapter` adapting different controller types to a uniform interface the `DispatcherServlet` can invoke.
- **Does Adapter violate the Single Responsibility Principle?** No — arguably it _enforces_ SRP by isolating interface-translation logic into its own class instead of letting client code handle both its own logic and interface conversion.

---

## Related Notes

- [[decorator]]
- [[facade]]
- [[bridge]]