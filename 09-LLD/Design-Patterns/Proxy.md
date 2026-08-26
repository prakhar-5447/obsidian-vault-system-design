# Proxy Pattern

## Intent

Provide a surrogate or placeholder for another object to **control access to it**. The Proxy implements the same interface as the real object, letting it stand in for that object transparently, while adding its own logic before/after delegating to it.

---

## Problem

Sometimes you want to add behavior **around** access to an object — controlling, deferring, logging, securing, or caching that access — without:

- Modifying the real object's own code (which may be off-limits, generated, or shared across many use cases that don't all need the extra behavior)
- Forcing every caller to remember to apply that logic themselves each time they use the object (e.g. every caller manually checking permissions before calling a method)
- Paying the cost of creating/loading an expensive object before it's actually needed

You need something that **looks and behaves like the real object** from the client's point of view, but actually intercepts the calls first.

---

## Solution

Create a **Proxy** class that implements the **same interface** as the real object (the "Subject" interface), and holds a reference to the real object (the "RealSubject"). The client interacts with the Proxy exactly as it would with the RealSubject — the Proxy forwards (delegates) calls to the real object, but can add logic before or after that delegation: checking access, logging, lazily creating the real object, caching results, or forwarding across a network.

Because Proxy and RealSubject share a common interface, **the client doesn't need to know or care** whether it's talking to the real object or a proxy standing in for it.

- **Virtual Proxy** — defers creation of an expensive object until it's actually needed (lazy loading)
- **Protection Proxy** — checks permissions/access rights before allowing a call to reach the real object
- **Remote Proxy** — represents an object that lives in a different address space/process/machine, hiding the network call behind a normal-looking local method call
- **Caching Proxy** — caches results of expensive calls to the real object, serving repeated requests from the cache instead

---

## Structure

```text
┌────────────┐        ┌───────────────────┐
│   Client   │───────▶│ Subject (interface) │
│            │        │  + request()          │
└────────────┘        └─────────┬─────────┘
                                 │ implemented by
                    ┌────────────┴────────────┐
                    ▼                          ▼
          ┌──────────────────┐      ┌──────────────────┐
          │      Proxy        │      │   RealSubject     │
          │ - realSubject      │─────▶│  + request() {     │
          │ + request() {       │      │    // actual work   │
          │    // pre-logic     │      │  }                   │
          │    realSubject      │      └──────────────────┘
          │      .request()     │
          │    // post-logic    │
          │  }                   │
          └──────────────────┘

Client only knows about the Subject interface.
Proxy intercepts calls, optionally does work, then delegates to RealSubject (which it may create lazily).
```

---

## Example

```text
// Subject interface — shared by Proxy and RealSubject
interface Image {
    void display();
}

// RealSubject — the expensive object we want to control access to
class HighResolutionImage implements Image {
    private final String filename;

    HighResolutionImage(String filename) {
        this.filename = filename;
        loadFromDisk(); // expensive operation happens at construction
    }

    private void loadFromDisk() {
        System.out.println("Loading " + filename + " from disk (expensive)...");
    }

    @Override
    public void display() {
        System.out.println("Displaying " + filename);
    }
}

// Virtual Proxy — defers creating the expensive RealSubject until display() is actually called
class ImageProxy implements Image {
    private final String filename;
    private HighResolutionImage realImage; // null until actually needed

    ImageProxy(String filename) {
        this.filename = filename; // cheap — no loading yet
    }

    @Override
    public void display() {
        if (realImage == null) {
            realImage = new HighResolutionImage(filename); // lazy creation, only on first display()
        }
        realImage.display();
    }
}

// Client — works with Image interface, unaware whether it's the proxy or the real thing
Image image = new ImageProxy("vacation.jpg"); // cheap — nothing loaded yet
System.out.println("Proxy created, image not loaded yet");

image.display(); // "Loading vacation.jpg from disk..." then "Displaying vacation.jpg" — first call
image.display(); // "Displaying vacation.jpg" only — real image already loaded, reused
```

---

## When to Use

- **Lazy initialization (Virtual Proxy)** — the real object is expensive to create, and you want to defer that cost until it's genuinely needed
- **Access control (Protection Proxy)** — different clients should have different permissions to call the real object's methods, and you want that check centralized rather than duplicated in the real object or scattered across callers
- **Remote access (Remote Proxy)** — the real object lives elsewhere (different process, server, service) — the Proxy hides the network call/serialization behind a normal-looking local interface (this is exactly what [[grpc]] client stubs do)
- **Caching (Caching Proxy)** — you want to cache the results of expensive calls to the real object, transparently to the client (conceptually related to [[caching]], applied at the object-interface level)
- **Logging/monitoring/auditing** — you want to record every call to an object without modifying the object itself

---

## When NOT to Use

- Adding a Proxy just to "future-proof" access to a simple, cheap object with no real access-control, laziness, or remoting need — unnecessary indirection for no actual benefit
- The extra layer of indirection meaningfully hurts performance in a latency-critical path, and the Proxy's added logic isn't actually needed there
- A simpler mechanism (e.g. a language-level property/getter, a decorator, or straightforward dependency injection) already solves the same problem with less structure

---

## Trade-offs

|Pros|Cons|
|---|---|
|Controls access to the real object without modifying its code|Adds a layer of indirection — every call goes through the Proxy first, adding some overhead|
|Client code is unaware it's talking to a Proxy — fully substitutable via the shared interface|Response from the real object can be delayed if the Proxy does significant work before delegating (e.g. lazy creation on first call)|
|Enables lazy loading of expensive resources, improving startup/initial load performance|Can introduce subtle bugs if the Proxy's behavior (e.g. caching) becomes stale or inconsistent with the real object's actual current state|
|Centralizes cross-cutting concerns (access control, caching, logging) in one place instead of duplicating them at every call site|Adds a new class per RealSubject type, increasing the number of types in the system|

---

## Interview Questions

- **How is Proxy different from Decorator?** Both wrap another object behind the same interface, and structurally look almost identical. The difference is intent: **Proxy controls access** to the real object (deciding whether/when/how the call reaches it — e.g. permission checks, lazy creation, remoting), while **Decorator adds new behavior/responsibilities** to an object while still always delegating to it (e.g. adding logging, formatting, or extra functionality on top of every call, with the wrapped object always fully accessible and used).
- **How is Proxy different from Adapter?** Adapter changes an object's **interface** to match what the client expects (bridging incompatible interfaces); Proxy keeps the **same interface** as the real object and controls access to it — the client-facing interface doesn't change at all with Proxy.
- **What's a real-world example of a Remote Proxy?** [[grpc]] and RPC framework client stubs are Remote Proxies — the generated client code looks like a normal local method call (`stub.getUser(id)`), but the Proxy actually serializes the request, sends it over the network, waits for a response, and deserializes it — the caller never touches sockets or serialization directly.
- **How would you implement a caching proxy, and what's the risk?** The Proxy checks an internal cache before delegating to the RealSubject; if present, returns the cached result directly. The risk is the same as any cache: staleness if the RealSubject's underlying data changes and the Proxy's cache isn't invalidated (see [[cache-invalidation]]) — the Proxy needs a strategy for expiring or refreshing cached results, not just an unconditional cache-forever policy.
- **Can multiple types of Proxy be combined?** Yes — e.g. a Remote Proxy can also include caching (to avoid a network round trip for repeated identical requests) and access control (to reject unauthorized calls before even attempting the network call) — real-world proxies often combine several of the "textbook" proxy categories in one implementation.
- **Is a Java dynamic proxy or Python's `__getattr__`-based proxying still "the Proxy pattern"?** Yes — the GoF description is about structure and intent (same interface, controlled access, delegation), not about how the language implements it. Dynamic proxies (reflection-based, generated at runtime) achieve the same pattern without hand-writing a wrapper class for every RealSubject type.

---

## Related Notes

- [[singleton]]
- [[caching]]
- [[cache-invalidation]]
- [[grpc]]
- [[command]]