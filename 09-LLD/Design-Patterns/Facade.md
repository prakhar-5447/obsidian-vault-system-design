# Facade Pattern

## Intent

Provide a unified, simplified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes a complex subsystem easier to use, without hiding the subsystem's full power from clients who still need direct access to it.

---

## Problem

A subsystem often grows to have many classes, each with their own responsibilities, dependencies, and initialization order (e.g. compiling and running code involves a lexer, parser, optimizer, and code generator; placing an order involves inventory, payment, shipping, and notification services), and:

- A typical client only wants to accomplish a **high-level task** ("compile this file", "place this order") — it shouldn't need to know the correct sequence of calls across half a dozen subsystem classes to do it correctly
- Every client that needs the subsystem ends up **duplicating** the same orchestration logic (which classes to call, in what order, with what glue code in between)
- The subsystem's classes become tightly coupled to every client that uses them directly — any internal restructuring of the subsystem risks breaking many unrelated client call sites
- New team members face a steep learning curve just to perform a common, simple task, because the subsystem's full complexity is exposed with no simplified entry point

---

## Solution

Introduce a **Facade** class that sits in front of the subsystem, exposing a small number of high-level methods that internally coordinate the necessary calls to the subsystem's classes in the correct order. Clients call the Facade for common tasks instead of talking to the subsystem directly — the Facade doesn't add new functionality, it just **simplifies and organizes access** to functionality that already exists.

Critically, the Facade does **not** prevent clients from bypassing it and using subsystem classes directly for advanced or unusual needs — it's an additional, optional simplified entry point, not a wall sealing the subsystem off.

- **Facade** — offers simple, high-level methods; internally delegates to and coordinates the subsystem classes
- **Subsystem classes** — continue to do the real work, unaware the Facade exists, and remain independently usable if a client needs finer-grained control

---

## Structure

```text
┌────────────┐        ┌──────────────────────┐
│  Client    │───────▶│       Facade           │
│            │        │ + simpleOperation() {   │
└────────────┘        │    subsystemA.step1()   │
                       │    subsystemB.step2()   │
     Client can also   │    subsystemC.step3()   │
     bypass Facade and │  }                       │
     call subsystem    └──────────┬───────────┘
     classes directly              │ coordinates
     if it needs to     ┌──────────┼──────────┐
                         ▼          ▼           ▼
                 ┌──────────┐┌──────────┐┌──────────┐
                 │Subsystem A││Subsystem B││Subsystem C│
                 └──────────┘└──────────┘└──────────┘

Facade doesn't add new capability — it just packages existing subsystem calls
into a simpler, higher-level operation for common use cases.
```

---

## Example

```text
// Subsystem classes — each with its own responsibility, unaware of each other or the Facade
class InventoryService {
    boolean reserve(String itemId, int qty) {
        System.out.println("Reserved " + qty + " of " + itemId);
        return true;
    }
}

class PaymentService {
    boolean charge(String customerId, double amount) {
        System.out.println("Charged " + customerId + " $" + amount);
        return true;
    }
}

class ShippingService {
    void scheduleShipment(String itemId, String address) {
        System.out.println("Scheduled shipment of " + itemId + " to " + address);
    }
}

class NotificationService {
    void sendConfirmation(String customerId) {
        System.out.println("Sent order confirmation to " + customerId);
    }
}

// Facade — coordinates the subsystem for the common "place an order" task
class OrderFacade {
    private final InventoryService inventory = new InventoryService();
    private final PaymentService payment = new PaymentService();
    private final ShippingService shipping = new ShippingService();
    private final NotificationService notification = new NotificationService();

    void placeOrder(String customerId, String itemId, int qty, double amount, String address) {
        if (inventory.reserve(itemId, qty)) {
            if (payment.charge(customerId, amount)) {
                shipping.scheduleShipment(itemId, address);
                notification.sendConfirmation(customerId);
            }
        }
    }
}

// Client — one call instead of manually orchestrating four subsystem classes
OrderFacade orderFacade = new OrderFacade();
orderFacade.placeOrder("cust123", "item456", 2, 49.99, "221B Baker Street");
// Reserved 2 of item456
// Charged cust123 $49.99
// Scheduled shipment of item456 to 221B Baker Street
// Sent order confirmation to cust123

// Advanced client can still bypass the Facade for finer control if needed:
new InventoryService().reserve("item456", 5); // direct subsystem access still works
```

---

## When to Use

- You want to provide a **simple entry point** for the most common use cases of a complex subsystem, without removing access to the subsystem's full capabilities for advanced clients
- You want to **decouple client code from subsystem internals** — clients depend on the Facade's stable, high-level interface rather than on many subsystem classes directly, so the subsystem can be refactored internally with minimal impact on clients
- You're introducing a third-party library or legacy subsystem into your codebase and want to wrap its awkward or overly detailed API behind something that fits your application's needs more naturally
- You want to **layer** a system — each layer exposes a Facade to the layer above it, reducing cross-layer coupling

---

## When NOT to Use

- The subsystem is already simple enough that a Facade would just be a thin, pointless pass-through with no real simplification
- Different clients need meaningfully different combinations/orderings of subsystem calls — forcing them all through one fixed Facade method can be more limiting than helpful; consider exposing subsystem classes directly, or several smaller Facades for different use cases
- You need the Facade to **also** add genuinely new behavior/logic beyond orchestration — at that point you may actually want a proper service/domain layer, not just a Facade (a Facade is meant to simplify access, not house significant new business logic)

---

## Trade-offs

|Pros|Cons|
|---|---|
|Simplifies common use cases into a single, high-level call|Can become a "god object" if too much responsibility gets piled into it over time|
|Decouples clients from subsystem internals — subsystem can be refactored with less risk of breaking clients|Doesn't prevent tight coupling between the Facade itself and every subsystem class it orchestrates|
|Reduces duplicated orchestration logic across many clients|If overused, can hide too much and make it harder for clients to understand what's actually happening under the hood|
|Doesn't restrict access — advanced clients can still use subsystem classes directly|Adding a Facade doesn't reduce the subsystem's actual complexity, only the _client-facing_ complexity — the subsystem itself is unchanged|

---

## Interview Questions

- **How is Facade different from Adapter?** Adapter's purpose is to make **one existing interface compatible** with what a client expects (translating one interface into another, usually one-to-one) — it's about interface _conversion_. Facade's purpose is to **simplify access to a whole subsystem** of multiple classes by exposing a smaller, higher-level interface — it's about interface _simplification and orchestration_, and typically wraps many classes at once rather than adapting one.
- **How is Facade different from Proxy?** [[proxy|Proxy]] implements the **same interface** as the object it wraps and controls access to that **one specific object** (lazy loading, permission checks, remoting). Facade provides a **new, simplified interface** in front of **multiple** subsystem classes — it's not about controlling access to one object, it's about making a whole subsystem easier to consume.
- **Does using a Facade mean clients can never access the subsystem directly?** No — this is a common misconception. Facade is meant to be an **additional, optional** simplified entry point for common cases; it doesn't hide or lock away the subsystem. Clients needing finer control are free to bypass the Facade and use subsystem classes directly.
- **Can you have multiple Facades over the same subsystem?** Yes — different Facades can expose different simplified views tailored to different client needs (e.g. an `AdminOrderFacade` with more granular control vs a `CustomerOrderFacade` with just `placeOrder()`), all sitting in front of the same underlying subsystem classes.
- **How does Facade help with layered architecture?** Each architectural layer can expose a Facade as its public entry point to the layer above, so upper layers depend only on that stable Facade interface rather than reaching into the lower layer's internal classes directly — this limits the blast radius when internals of a layer are refactored.
- **Where have you seen Facade used in real systems/frameworks?** An ORM's high-level `save()`/`find()` API sitting in front of connection management, SQL generation, and transaction handling; a payment SDK's single `charge()` method that internally handles authentication, retries, and multiple downstream calls; a logging framework's simple `logger.info()` call that hides formatter, appender, and handler configuration underneath.

---

## Related Notes

- [[proxy]]
- [[command]]
- [[strategy]]