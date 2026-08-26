# Chain of Responsibility Pattern

## Intent

Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects together and pass the request along the chain until an object handles it (or the chain ends).

---

## Problem

A request needs to be processed by **one of several possible handlers**, but the sender shouldn't need to know:

- **Which** specific handler will end up processing it
- **How many** handlers exist, or their order
- Whether it takes **one** handler or a **sequence** of handlers to fully process it (e.g. logging, then auth check, then validation, then the actual business logic)

A naive approach hardcodes a big `if/else` or `switch` in the sender to pick the right handler — this couples the sender to every possible handler's concrete type, and adding a new handler means modifying that conditional every time, violating Open/Closed.

---

## Solution

Give each handler a reference to the **next** handler in the chain. Each handler decides independently whether it can process the request:

- **Process it and stop** — the chain ends here
- **Process it and pass it on anyway** — useful when multiple handlers should each get a chance (e.g. logging middleware that logs _and_ forwards)
- **Not process it, pass it to the next handler** — delegate down the chain
- **Chain ends with no handler processing it** — the client defines a default/fallback behavior for this case

The sender only ever talks to the **first** handler in the chain — it has no idea how many handlers exist or which one (if any) actually processes the request.

---

## Structure

```text
┌────────────┐        ┌──────────────────────┐
│   Client   │───────▶│   Handler (interface) │
└────────────┘        │  + setNext(Handler)   │
                       │  + handle(Request)    │
                       └───────────┬───────────┘
                                   │ implemented by
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                     ▼
     ┌────────────────┐  ┌────────────────┐   ┌────────────────┐
     │ ConcreteHandlerA│─▶│ ConcreteHandlerB│─▶ │ ConcreteHandlerC│─▶ (end)
     │ handle():        │  │ handle():        │   │ handle():        │
     │  if canHandle →  │  │  if canHandle →  │   │  if canHandle →  │
     │    process        │  │    process        │   │    process        │
     │  else             │  │  else             │   │  else             │
     │    next.handle()  │  │    next.handle()  │   │    (chain ends)   │
     └────────────────┘  └────────────────┘   └────────────────┘

Client only holds a reference to the FIRST handler.
Each handler holds a reference to the NEXT handler.
Request flows down the chain until someone handles it (or it ends).
```

---

## Example

```text
// Handler — abstract base defining the chaining mechanism
abstract class SupportHandler {
    protected SupportHandler next;

    SupportHandler setNext(SupportHandler next) {
        this.next = next;
        return next; // enables fluent chain-building: h1.setNext(h2).setNext(h3)
    }

    void handle(SupportTicket ticket) {
        if (next != null) {
            next.handle(ticket); // default: pass along if this handler doesn't override
        } else {
            System.out.println("No handler could resolve ticket: " + ticket.getIssue());
        }
    }
}

// Concrete handlers — each decides independently whether it can process the request
class Level1SupportHandler extends SupportHandler {
    @Override
    void handle(SupportTicket ticket) {
        if (ticket.getSeverity() <= 2) {
            System.out.println("Level1 resolved: " + ticket.getIssue());
        } else {
            super.handle(ticket); // delegate to next in chain
        }
    }
}

class Level2SupportHandler extends SupportHandler {
    @Override
    void handle(SupportTicket ticket) {
        if (ticket.getSeverity() <= 5) {
            System.out.println("Level2 resolved: " + ticket.getIssue());
        } else {
            super.handle(ticket);
        }
    }
}

class ManagerHandler extends SupportHandler {
    @Override
    void handle(SupportTicket ticket) {
        System.out.println("Manager escalation resolved: " + ticket.getIssue());
        // manager is the last resort — always handles, no super.handle() call
    }
}

// Client — builds the chain once, then only talks to the first handler
SupportHandler chain = new Level1SupportHandler();
chain.setNext(new Level2SupportHandler()).setNext(new ManagerHandler());

chain.handle(new SupportTicket("password reset", 1));   // Level1 resolves it
chain.handle(new SupportTicket("data breach", 9));       // escalates to Manager
```

---

## When to Use

- More than one object **may** handle a request, and the handler isn't known until runtime — e.g. support ticket escalation by severity, middleware pipelines (auth → logging → validation → business logic)
- You want to issue a request **without specifying the receiver explicitly** — decoupling sender from every possible handler's concrete class
- The set of handlers and their order should be **configurable at runtime** rather than hardcoded — new handlers can be inserted without modifying the sender or other handlers
- Multiple handlers might need to act on the **same** request in sequence (e.g. HTTP middleware/filter chains, where each layer runs and forwards)

---

## When NOT to Use

- There's exactly **one** handler for a given request type and it's always known — a direct call is simpler and clearer than building a chain of one
- The chain would become very long and **hard to debug** — tracing which handler actually processed (or silently dropped) a request across a deep chain can get confusing without good logging at each link
- You need a **guarantee** that some handler will always process the request — Chain of Responsibility doesn't inherently guarantee this; you must explicitly design a terminal fallback handler (as `ManagerHandler` does above), or the request can silently fall through unhandled
- Handlers must be evaluated in **parallel** or need shared coordinated state beyond pass-through — this pattern is inherently sequential and stateless between handlers by design

---

## Trade-offs

|Pros|Cons|
|---|---|
|Decouples sender from receivers — sender only knows the chain's entry point|Request may silently go unhandled if no handler processes it and no fallback exists|
|Open/Closed — new handlers can be added without modifying existing ones or the sender|Can be harder to debug/trace than a direct call — the actual handler isn't obvious from the call site|
|Handler order is flexible/configurable at runtime|If the chain gets long, request processing can incur more indirection/overhead per request|
|Naturally models middleware/pipeline-style processing (logging, auth, validation layers)|No built-in enforcement that the chain is well-formed (e.g. must remember to call `setNext` for every link)|

---

## Interview Questions

- **How is Chain of Responsibility different from Decorator?** Both wrap/link objects together, but Decorator's whole point is that **every** decorator in the stack processes the request (adding behavior), while Chain of Responsibility's point is that typically **only one** handler ends up processing it (others just pass through). Decorator is about layering behavior; Chain of Responsibility is about routing to the right handler.
- **How does this differ from just using a `switch` statement to pick a handler?** A switch hardcodes every case in one place, coupling the caller to all possible handler types and violating Open/Closed when a new case is added. Chain of Responsibility lets each handler be a self-contained class, and new handlers are added by wiring them into the chain, not by editing existing code.
- **What happens if no handler in the chain can process the request?** By default, nothing — the request silently falls through. You must design an explicit terminal/fallback handler (or throw/log at the end of the chain) to guarantee some outcome, as this pattern doesn't provide that guarantee automatically.
- **Where have you seen this pattern in real frameworks?** Servlet Filters / Express.js middleware / most HTTP middleware pipelines (each layer processes and calls `next()`); Java's `java.util.logging.Logger` (log handlers can pass a record up to a parent logger); exception handling itself is conceptually a chain (a try/catch that doesn't match rethrows up the call stack).
- **Can multiple handlers process the same request in this pattern?** Yes — nothing prevents a handler from processing the request _and_ still calling `next.handle()` to forward it further (e.g. a logging handler that always logs and always forwards) — the "stop after the first handler" behavior shown in the example is a common convention, not a hard requirement of the pattern itself.

---

## Related Notes

- [[decorator]]
- [[observer]]
- [[strategy]]