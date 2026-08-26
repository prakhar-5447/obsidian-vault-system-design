# State Pattern

## Intent

Allow an object to alter its behavior when its internal state changes. The object will appear to change its class — behavior is delegated to a State object, and swapping that State object changes what the object does, without changing the object's own code.

---

## Problem

An object's behavior needs to change based on **which state it's currently in** (e.g. a document being Draft/Moderation/Published, a media player being Playing/Paused/Stopped, a traffic light being Red/Yellow/Green), and:

- A naive implementation uses a state field (`enum` or flag) plus large conditional blocks (`if state == X ... else if state == Y ...`) scattered across every method that behaves differently per state
- These conditionals grow unwieldy as more states or methods are added, and duplicate the same state-check logic in multiple places
- Adding a new state means hunting down and editing every conditional block across the class — a clear Open/Closed violation
- It's easy to end up with **invalid state transitions** because the transition logic isn't centralized anywhere

---

## Solution

Extract each state into its own class implementing a common **State** interface, with one method per state-dependent behavior. The **Context** class holds a reference to its _current_ State object and delegates all state-dependent behavior to it, rather than branching on a state flag internally.

Each ConcreteState implements the behavior appropriate for that state, and — critically — **is often responsible for triggering the transition to the next state** (by telling the Context to swap its current State object), which is what distinguishes State from a Strategy that just picks an algorithm.

- **Context** — holds a reference to the current State, delegates behavior to it, exposes a way to change the current State
- **State (interface)** — declares the operations whose behavior varies per state
- **ConcreteState** — implements the behavior for one specific state, and often decides/triggers the next state

---

## Structure

```text
┌────────────┐        ┌──────────────────┐
│  Context   │───────▶│  State (interface) │
│ - state    │        │  + handle()          │
│ + setState()│        └─────────┬─────────┘
│ + request(){│                  │ implemented by
│   state.handle(this)│          ▼
│  }          │        ┌──────────────────┐
└────────────┘         │  ConcreteStateA   │
                        │  + handle(context){│
                        │    // do work,      │
                        │    // then maybe:   │
                        │    context.setState(│
                        │      new StateB())  │
                        │  }                   │
                        └──────────────────┘
                        ┌──────────────────┐
                        │  ConcreteStateB   │
                        │  + handle(context){ ... }│
                        └──────────────────┘

Context delegates to its current State object.
ConcreteStates often transition the Context to a different State as part of handling a request.
```

---

## Example

```text
// State interface
interface OrderState {
    void next(OrderContext context);
    String getName();
}

// Concrete States — each knows what comes next
class PlacedState implements OrderState {
    @Override
    public void next(OrderContext context) {
        System.out.println("Order placed -> shipping");
        context.setState(new ShippedState());
    }

    @Override
    public String getName() { return "PLACED"; }
}

class ShippedState implements OrderState {
    @Override
    public void next(OrderContext context) {
        System.out.println("Order shipped -> delivered");
        context.setState(new DeliveredState());
    }

    @Override
    public String getName() { return "SHIPPED"; }
}

class DeliveredState implements OrderState {
    @Override
    public void next(OrderContext context) {
        System.out.println("Order already delivered — no further transitions");
        // terminal state: no-op, or could throw to signal an invalid transition
    }

    @Override
    public String getName() { return "DELIVERED"; }
}

// Context — delegates behavior + transitions to whichever State it currently holds
class OrderContext {
    private OrderState state = new PlacedState(); // initial state

    void setState(OrderState state) {
        this.state = state;
    }

    void advance() {
        state.next(this); // delegates — Context has no "if state == X" logic at all
    }

    String currentState() {
        return state.getName();
    }
}

// Client
OrderContext order = new OrderContext();
System.out.println(order.currentState()); // PLACED
order.advance();                           // Order placed -> shipping
System.out.println(order.currentState()); // SHIPPED
order.advance();                           // Order shipped -> delivered
System.out.println(order.currentState()); // DELIVERED
order.advance();                           // Order already delivered — no further transitions
```

---

## When to Use

- An object's behavior depends heavily on its current state, and that behavior differs across **many methods** — State centralizes each state's behavior into one cohesive class instead of scattering conditionals across the object
- You have a **well-defined set of states with explicit transitions** between them (order lifecycle, connection states, workflow/approval stages, game character states) — modeling this as a finite state machine
- You want adding a new state to be a matter of **adding a new class**, not editing every existing conditional block (Open/Closed)
- You want to eliminate bugs from invalid or inconsistent state-checking logic duplicated across methods

---

## When NOT to Use

- The object has only 2-3 simple states with minimal behavior differences — a boolean flag or enum with a small conditional may be perfectly readable and simpler than a full class hierarchy
- States rarely or never change once set (this is closer to configuration/Strategy than a lifecycle) — consider [[strategy]] instead
- The overhead of creating a new object on every transition is a genuine performance concern in a very hot path (rare, but worth flagging for extremely latency-sensitive systems) — a flyweight/shared State instance can mitigate this since ConcreteStates are often stateless themselves

---

## Trade-offs

|Pros|Cons|
|---|---|
|Eliminates large conditional blocks checking the current state|Increases the number of classes — one per state|
|Each state's behavior and transitions are localized in one class, easier to understand and modify|Transition logic spread across ConcreteState classes can make the overall state machine harder to see at a glance (no single place lists all transitions)|
|Adding a new state doesn't require editing existing state classes (Open/Closed)|If states need to share some behavior, requires a common base class or careful interface design to avoid duplication|
|Prevents invalid ad-hoc state manipulation — transitions only happen through defined State methods|Overkill for objects with very few, simple states|

---

## Interview Questions

- **How is State different from Strategy?** Structurally, they're nearly identical (Context delegates to an interchangeable object). The difference is intent: in [[strategy]], the **client** chooses which algorithm/strategy to inject, and the strategies are typically unaware of one another and don't trigger swaps themselves. In State, the **Context's current behavior set naturally transitions** from one State to another, often triggered by the ConcreteState itself — the states model a connected lifecycle, not independent interchangeable algorithms.
- **Who should be responsible for transitions — the Context or the ConcreteState?** Either is valid, but letting the **ConcreteState** decide the next state (as in the example above) keeps all the transition logic for "what comes after PLACED" together with the PLACED state itself, rather than in the Context, which would otherwise need to know about every state's transition rules — better cohesion, closer to Open/Closed.
- **How would you prevent an invalid transition (e.g. going from DELIVERED back to PLACED)?** The ConcreteState for a terminal or restricted state simply doesn't implement a path to that transition — e.g. `DeliveredState.next()` above is a no-op. More strictly, it could throw an `IllegalStateException` to make invalid transitions loud rather than silent.
- **Is the State pattern the same as a Finite State Machine (FSM)?** State pattern is one common **object-oriented implementation** of an FSM — it's not the only way to build one (table-driven FSMs, using a map of `(state, event) -> nextState`, are another common approach, often simpler for very large state machines with regular structure). State pattern trades some of that table's compactness for more expressive, testable, per-state behavior encapsulated in real classes.
- **Can ConcreteState instances be shared/reused across multiple Context objects?** Yes, if the ConcreteState classes are stateless (hold no instance fields specific to one Context) — in that case, a single shared instance of each ConcreteState can be reused (often as singletons) across many Contexts, avoiding unnecessary object churn on every transition.
- **Where have you seen State used in real systems?** TCP connection state machines (Listen, SynSent, Established, Closed, etc. — see [[tcp]]); order/shipment lifecycle systems; media player controls (Playing/Paused/Stopped, each interpreting "press play" differently); parser/lexer implementations that behave differently depending on what token type they're currently reading; UI components like a multi-step form wizard.

---

## Related Notes

- [[strategy]]
- [[command]]
- [[observer]]
- [[tcp]]