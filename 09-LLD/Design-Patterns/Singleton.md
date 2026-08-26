# Singleton Pattern

## Intent

Ensure a class has only **one instance**, and provide a single, well-known global point of access to it.

---

## Problem

Some things in a system should genuinely exist **exactly once** — a configuration manager, a connection pool, a logging service, a hardware interface — and:

- Creating multiple instances would waste resources (e.g. multiple connection pools each opening their own connections) or cause inconsistent state (e.g. two logger instances writing to two different files)
- Passing a shared instance through every constructor across the codebase is tedious and pollutes APIs that don't otherwise need to know about it
- You need a **single, predictable access point** that any part of the codebase can reach, without a global variable's lack of control (no lazy initialization, no protection against accidental reassignment)

---

## Solution

Make the class responsible for **controlling its own instantiation**: a private constructor prevents anyone else from creating new instances, and a static method (commonly `getInstance()`) returns the one shared instance — creating it on first access (lazy initialization) if it doesn't exist yet.

- **Eager initialization** — the instance is created immediately when the class is loaded, whether or not it's ever used
- **Lazy initialization** — the instance is only created the first time `getInstance()` is called
- **Thread-safe lazy initialization** — lazy initialization guarded against race conditions when multiple threads call `getInstance()` simultaneously (see the double-checked locking example below)

---

## Structure

```text
┌──────────────────────────────┐
│          Singleton           │
├──────────────────────────────┤
│ - static instance: Singleton │
│ - Singleton() { }             │  ← private constructor,
│                                │     no one else can "new" it
│ + static getInstance():       │
│     Singleton {               │
│       if (instance == null)   │
│         instance = new ...()  │
│       return instance         │
│   }                            │
└──────────────────────────────┘

All client code calls Singleton.getInstance() — never "new Singleton()".
The class itself is the only place that can create an instance of itself.
```

---

## Example

```text
// Thread-safe lazy Singleton (double-checked locking)
class ConnectionPool {
    private static volatile ConnectionPool instance;
    private final List<Connection> connections;

    // private constructor — prevents "new ConnectionPool()" from outside
    private ConnectionPool() {
        connections = createInitialConnections();
        System.out.println("ConnectionPool created");
    }

    public static ConnectionPool getInstance() {
        if (instance == null) {                     // first check (no lock, fast path)
            synchronized (ConnectionPool.class) {
                if (instance == null) {              // second check (inside lock, avoids race)
                    instance = new ConnectionPool();
                }
            }
        }
        return instance;
    }

    Connection getConnection() {
        return connections.get(0); // simplified
    }
}

// Client — always goes through getInstance(), never constructs directly
ConnectionPool pool1 = ConnectionPool.getInstance(); // "ConnectionPool created" printed here
ConnectionPool pool2 = ConnectionPool.getInstance(); // no print — same instance returned

System.out.println(pool1 == pool2); // true — genuinely the same object
```

**Why `volatile` and double-checked locking:** without `volatile`, a thread could see a partially-constructed instance due to instruction reordering by the compiler/CPU; the first (unsynchronized) null-check is a performance optimization so that after the instance exists, subsequent calls don't pay the cost of acquiring a lock at all.

---

## When to Use

- Exactly one instance of a class must exist for the lifetime of the application (a single shared configuration object, a single hardware/device interface, a single application-wide cache manager)
- You need controlled, lazy creation of an expensive resource that many parts of the system need access to, but shouldn't each create their own copy of
- You want a single, well-known access point without threading a reference through every constructor/method signature across the codebase

---

## When NOT to Use

- **Singleton is one of the most over-used and criticized patterns** — reach for it deliberately, not by default
- If the "single instance" requirement is really just a convenience, not a genuine architectural constraint — consider a dependency-injected shared instance instead (see trade-offs below)
- When it's used to avoid passing dependencies explicitly ("global variable in disguise") rather than because only one instance should truly ever exist — this creates hidden coupling that's hard to trace
- When you'll ever plausibly need **multiple** instances later (e.g. multi-tenant systems, or testing with isolated instances) — Singleton makes this a painful retrofit

---

## Trade-offs

|Pros|Cons|
|---|---|
|Guarantees exactly one instance exists, with controlled access|Introduces **global state** — any code anywhere can reach and mutate it, making data flow harder to trace|
|Lazy initialization avoids creating expensive resources until actually needed|**Makes unit testing harder** — tests can't easily substitute a mock/fake instance, since the class controls its own construction and code calls `getInstance()` directly rather than receiving a dependency|
|Single well-known access point, no need to pass references through every layer|Hides dependencies — a method calling `Singleton.getInstance()` doesn't declare that dependency in its signature, unlike a constructor parameter|
|Solves real problems for genuinely single-instance resources (e.g. a hardware interface)|Encourages tight coupling to a concrete implementation — hard to swap in a different implementation for a different context|
||In multi-threaded environments, needs careful synchronization to avoid two threads creating two "singleton" instances (see double-checked locking) — adds complexity|

**The dependency injection alternative:** in most modern codebases, the "there should be one instance" goal is achieved by having a DI container/framework create and manage a single instance (a "singleton-scoped" bean/service), and **inject** it wherever needed via the constructor — this achieves the same single-instance guarantee while keeping the dependency explicit and mockable in tests, avoiding most of Singleton's downsides. This is why many engineers consider the classic GoF Singleton pattern (with its own static `getInstance()`) something of an anti-pattern in modern application code, while still endorsing "singleton-scoped" objects managed by a framework.

---

## Interview Questions

- **Why is Singleton considered an anti-pattern by some engineers?** Because it introduces global mutable state and hidden dependencies, and makes unit testing significantly harder (you can't substitute a test double for a class that controls its own instantiation via a private constructor and static accessor). Most of Singleton's legitimate benefit (one shared instance) can be achieved via dependency injection instead, without those downsides.
- **How do you make a Singleton thread-safe?** Options include: double-checked locking with a `volatile` field (shown above); eager initialization (create the instance at class-load time, which is inherently thread-safe but loses laziness); or, in Java specifically, the "initialization-on-demand holder" idiom, which relies on the JVM's class-loading guarantees to get lazy + thread-safe initialization without explicit synchronization at all.
- **How does Singleton complicate unit testing?** Because code depends on `Singleton.getInstance()` directly rather than having the instance passed in, tests can't easily swap in a mock or a fresh instance per test — tests can end up sharing state through the singleton across test cases, causing subtle ordering-dependent test failures.
- **How is Singleton different from just using static methods/fields?** A Singleton is a genuine object instance (supports interfaces, inheritance, lazy construction, instance state) accessed through a controlled static accessor — whereas a purely static class has no instance identity at all, can't implement an interface, and can't be lazily/conditionally constructed the same way.
- **Can a Singleton be broken (i.e., can you accidentally get two instances)?** Yes — via reflection (forcibly calling the private constructor), deserialization (creating a new object from serialized bytes), or cloning (if `Clone()`/`clone()` isn't overridden to return the existing instance) — defensive Singletons in some languages guard against all three explicitly.
- **How would you refactor away from a Singleton if it's causing testing pain?** Introduce an interface for the singleton's behavior, have the "real" implementation be constructed once by a DI container and injected wherever needed, and let tests inject a mock/fake implementation of the same interface instead of reaching for the global instance.

---

## Related Notes

- [[proxy]]
- [[strategy]]
- [[command]]