# Observer Pattern

## Intent

Define a one-to-many dependency between objects so that when one object (the Subject) changes state, all its dependents (Observers) are notified and updated automatically. Observer decouples the object whose state changes from the objects that need to react to that change.

---

## Problem

You often need multiple parts of a system to **react** when something changes (a UI needs to refresh when data updates, a logging system needs to record every state change, several services need to react to the same event), and:

- Hardcoding direct calls from the Subject to every interested party tightly couples the Subject to specific concrete classes, and requires editing the Subject every time a new dependent is added
- The Subject shouldn't need to know **what** its dependents do with the notification, or even how many there are — only that they want to be told when something happens
- You want to be able to add or remove interested parties **at runtime**, without touching the Subject's code

---

## Solution

The **Subject** maintains a list of **Observers** and provides methods to `subscribe()`/`unsubscribe()` them. When the Subject's state changes, it iterates its list and calls `update()` (or similar) on every registered Observer — without knowing or caring what each Observer actually does in response.

Observers implement a common interface so the Subject can treat them uniformly, regardless of their concrete type. This is the core mechanism behind [[event-driven-architecture]] and [[pub-sub]] at the object level, before those patterns are extended across processes/network boundaries.

- **Subject** — holds the list of Observers, exposes subscribe/unsubscribe, notifies on state change
- **Observer (interface)** — declares the `update()` method the Subject calls
- **ConcreteSubject** — the actual object whose state changes are worth observing
- **ConcreteObserver** — reacts to the notification in its own way

**Push vs Pull notification:**

- **Push** — the Subject sends the changed data directly as arguments to `update()`
- **Pull** — the Subject just notifies that _something_ changed; Observers query the Subject afterward for whatever specific data they need

---

## Structure

```text
┌────────────────────┐        ┌──────────────────┐
│      Subject        │───────▶│ Observer (interface)│
│ - observers: List    │        │  + update(data)       │
│ + subscribe(o)        │        └─────────┬─────────┘
│ + unsubscribe(o)       │                  │ implemented by
│ + notify() {            │                  ▼
│    for o in observers:  │        ┌──────────────────┐
│      o.update(this.state)│        │ ConcreteObserverA │
│  }                        │        │  + update(data) { ... }│
└──────────┬───────────┘        └──────────────────┘
           │ extended by          ┌──────────────────┐
           ▼                      │ ConcreteObserverB │
┌────────────────────┐            │  + update(data) { ... }│
│  ConcreteSubject     │            └──────────────────┘
│ - state               │
│ + setState(s) {        │
│    state = s            │
│    notify()              │
│  }                        │
└────────────────────┘

Subject knows only the Observer interface — never concrete Observer types.
Observers can be added/removed at runtime without modifying the Subject.
```

---

## Example

```text
// Observer interface
interface Observer {
    void update(int temperature);
}

// Subject — maintains observers, notifies them on state change
class WeatherStation {
    private final List<Observer> observers = new ArrayList<>();
    private int temperature;

    void subscribe(Observer o) {
        observers.add(o);
    }

    void unsubscribe(Observer o) {
        observers.remove(o);
    }

    void setTemperature(int temp) {
        this.temperature = temp;
        notifyObservers(); // state changed — tell everyone who's listening
    }

    private void notifyObservers() {
        for (Observer o : observers) {
            o.update(temperature); // push notification with the new value
        }
    }
}

// Concrete Observers — each reacts differently, Subject doesn't know or care how
class PhoneDisplay implements Observer {
    @Override
    public void update(int temperature) {
        System.out.println("Phone display: " + temperature + "°C");
    }
}

class LoggingObserver implements Observer {
    @Override
    public void update(int temperature) {
        System.out.println("Logged temperature reading: " + temperature);
    }
}

// Client — wires Subject and Observers together
WeatherStation station = new WeatherStation();
Observer phone = new PhoneDisplay();
Observer logger = new LoggingObserver();

station.subscribe(phone);
station.subscribe(logger);

station.setTemperature(25);
// Phone display: 25°C
// Logged temperature reading: 25

station.unsubscribe(logger);
station.setTemperature(30);
// Phone display: 30°C     ← logger no longer notified
```

---

## When to Use

- A change to one object needs to trigger updates in an **unknown or variable number** of other objects, and you don't want the Subject hardcoded to know about each of them
- You want to add/remove interested parties **at runtime** without modifying the Subject's code (Open/Closed)
- You're implementing a UI/data-binding system, an event system, or any "something happened, react accordingly" workflow
- You want to decouple **what changed** from **what happens as a result** — the Subject shouldn't need to know or care what its Observers actually do

---

## When NOT to Use

- There's only ever one, fixed dependent that reacts to the change — a direct method call is simpler and easier to trace than the indirection of subscribe/notify
- Notification order matters and must be strictly guaranteed/coordinated — plain Observer doesn't inherently provide ordering guarantees across multiple Observers, and relying on registration order can be fragile
- The system has so many Observers, or Observers so expensive to notify, that synchronous notification becomes a performance bottleneck — at that point, an actual message queue/event bus (see [[message-queues]], [[pub-sub]]) with async delivery is more appropriate than an in-process Observer list
- Observers that fail or throw exceptions could disrupt the Subject's own logic if not carefully isolated — needs explicit handling (e.g. catching exceptions per-Observer so one failing Observer doesn't break notification to the rest)

---

## Trade-offs

|Pros|Cons|
|---|---|
|Decouples Subject from concrete Observer types — Subject only depends on the Observer interface|Notification order across multiple Observers is often unspecified/registration-order-dependent, which can be fragile if order matters|
|Observers can be added/removed dynamically at runtime|Can lead to unexpected update chains if Observers themselves trigger further state changes that notify other Observers (cascading updates, hard to trace)|
|Supports broadcast communication — one change, many independent reactions|If not carefully managed, forgetting to unsubscribe can cause memory leaks (Subject holds references to Observers that should have been garbage collected) — the classic "lapsed listener" problem|
|Enables loose coupling well-suited to event-driven and reactive designs|Synchronous notification can become a performance bottleneck if there are many Observers or slow ones — no built-in async delivery, ordering, or durability guarantees (unlike [[message-queues]])|

---

## Interview Questions

- **How is Observer different from Pub/Sub (as in [[pub-sub]])?** They're conceptually the same core idea (broadcast notification to interested parties), but Observer is typically an **in-process, tightly coupled** pattern — the Subject holds direct references to Observer objects and calls them synchronously. Pub/Sub (as typically implemented with message brokers) usually adds a **broker/mediator** in between (so publishers and subscribers don't hold direct references to each other at all), and often supports asynchronous, cross-process, or cross-network delivery — Observer is the simpler, foundational pattern that Pub/Sub systems build upon and extend.
- **What's the "lapsed listener" problem, and how do you avoid it?** If Observers subscribe to a long-lived Subject but are never explicitly unsubscribed (e.g. a UI component that's destroyed but forgot to unsubscribe from a global event source), the Subject keeps holding a reference to it, preventing garbage collection — a memory leak. Mitigations include explicit `unsubscribe()` calls in cleanup/destructor logic, or using weak references so the Subject doesn't prevent garbage collection of Observers no longer used elsewhere.
- **Should notify() be synchronous or asynchronous?** Classic Observer implementations are synchronous — the Subject blocks while each Observer processes the update, which is simple but means a slow Observer can slow down the entire notification (and by extension, the Subject's own operation). For expensive or numerous Observers, an async approach (dispatching update work to a thread pool, or moving to a real message queue as in [[message-queues]]) avoids blocking the Subject, at the cost of losing the simple, synchronous guarantee that all Observers have processed the update before the Subject's method returns.
- **How would you handle an Observer throwing an exception during notification?** Wrap each Observer's `update()` call individually in a try/catch within the notification loop, so one failing Observer doesn't prevent the remaining Observers from being notified, and doesn't propagate an unrelated Observer's failure back to whatever triggered the Subject's state change.
- **Is Observer related to Command?** Not directly, but they're often combined — a Command's `execute()` might, after performing its action, notify [[observer|Observers]] that the action occurred (e.g. a "SaveCommand" notifying a status-bar Observer that a save just happened) — the two patterns solve different problems and compose naturally.
- **Where have you seen Observer used in real systems/frameworks?** GUI event listeners (`addEventListener` in JavaScript, `ActionListener` in Java Swing); reactive programming libraries (RxJS, Reactive Streams) which are essentially a much richer, composable evolution of Observer; model-view frameworks where views subscribe to model changes; Redux/Vuex-style state management, where components subscribe to store changes.

---

## Related Notes

- [[command]]
- [[pub-sub]]
- [[event-driven-architecture]]
- [[message-queues]]