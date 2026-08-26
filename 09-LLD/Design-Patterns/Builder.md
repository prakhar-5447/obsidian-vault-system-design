# Builder Pattern

## Intent

Separate the construction of a complex object from its representation, so the same construction process can create different representations. Builder lets you construct an object **step by step**, producing different types/configurations of the object using the same building process, without a telescoping constructor or a giant parameter list.

---

## Problem

A class needs to be constructed with **many optional fields/parameters**, and a plain constructor approach breaks down:

- **Telescoping constructor anti-pattern** — overloaded constructors for every combination of optional params (`Pizza(size)`, `Pizza(size, cheese)`, `Pizza(size, cheese, pepperoni)`...) becomes unreadable and error-prone as fields grow
- **Setter-based construction** leaves the object in a **partially-constructed, mutable state** during setup, and gives no guarantee the object is complete/valid before it's used
- Some objects require **validation or specific construction order** across several steps — a single flat constructor call obscures that logic
- You may want to build **different representations** of a conceptually similar object (e.g. a `Report` built as a `PDFReport` vs `HTMLReport`) via the same step-by-step process

---

## Solution

Extract the object-construction logic into a separate **Builder** class that exposes fluent, chainable methods for setting each optional field, and a final `build()` method that returns the fully-constructed, **immutable** object.

- **Fluent Builder** (most common in practice) — each setter returns `this`, allowing method chaining; a `build()` method performs final validation and constructs the immutable target object
- **Director** (classic GoF variant) — an optional separate class that knows specific _sequences_ of builder calls to produce known configurations (e.g. `buildSportsCarConfig()`), useful when the same builder is reused to produce several standard "recipes" — often omitted in modern usage where the fluent builder alone is enough

---

## Structure

```text
┌────────────┐        ┌───────────────────────┐
│  Director  │ (opt.) │       Builder         │
│  (knows    │───────▶│  + setPartA()         │
│  recipes)  │        │  + setPartB()         │
└────────────┘        │  + setPartC()         │
                       │  + build(): Product   │
                       └───────────┬───────────┘
                                   │ constructs
                                   ▼
                       ┌───────────────────────┐
                       │        Product         │
                       │  (immutable, complete  │
                       │   once built)          │
                       └───────────────────────┘

Client calls builder methods in a chain, then build().
Builder accumulates state; Product is only created at the end.
```

---

## Example

```text
// Product — immutable once constructed, no public constructor
class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final String body;
    private final int timeoutMs;

    // private constructor — only the Builder can create instances
    private HttpRequest(Builder builder) {
        this.url = builder.url;
        this.method = builder.method;
        this.headers = builder.headers;
        this.body = builder.body;
        this.timeoutMs = builder.timeoutMs;
    }

    static Builder newBuilder(String url) {
        return new Builder(url);
    }

    // Static nested Builder class
    static class Builder {
        private final String url;               // required
        private String method = "GET";           // sensible defaults
        private Map<String, String> headers = new HashMap<>();
        private String body = null;
        private int timeoutMs = 5000;

        private Builder(String url) {
            this.url = url; // required field passed in constructor, not a setter
        }

        Builder method(String method) {
            this.method = method;
            return this; // enables chaining
        }

        Builder header(String key, String value) {
            this.headers.put(key, value);
            return this;
        }

        Builder body(String body) {
            this.body = body;
            return this;
        }

        Builder timeoutMs(int timeoutMs) {
            this.timeoutMs = timeoutMs;
            return this;
        }

        HttpRequest build() {
            // validation happens here, before the object ever exists
            if ("POST".equals(method) && body == null) {
                throw new IllegalStateException("POST requests require a body");
            }
            return new HttpRequest(this);
        }
    }
}

// Client usage — fluent, readable, no telescoping constructor
HttpRequest request = HttpRequest.newBuilder("https://api.example.com/orders")
        .method("POST")
        .header("Content-Type", "application/json")
        .body("{\"item\": \"widget\"}")
        .timeoutMs(3000)
        .build();
```

---

## When to Use

- An object has **many optional parameters/fields**, and most combinations don't need every one set
- You want the constructed object to be **immutable** once built, but still need multi-step, flexible construction
- Construction involves **validation logic** that should run once, at the end, rather than being scattered across setters
- You want a **readable, self-documenting** construction call (`.method("POST").body(...).build()` reads far better than a 7-argument constructor call where you have to count positions)
- You need to produce **several representations** of a conceptually similar object via a shared step-by-step process (paired with a Director)

---

## When NOT to Use

- The object has **few fields** (2–3), all typically required — a plain constructor is simpler and doesn't need the extra Builder class boilerplate
- You don't need immutability and mutation via simple setters is acceptable — Builder adds ceremony for no real benefit in that case
- Performance-critical, high-allocation-frequency code where the extra Builder object allocation (even if short-lived) matters — rare, but worth naming as an edge case
- In languages with **named/default parameters** (Python, Kotlin, C#) the need for Builder is much lower — a constructor or factory function with keyword defaults often achieves the same readability without a separate class

---

## Trade-offs

| Pros                                                                                        | Cons                                                                                             |
| ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Avoids telescoping constructors and long positional parameter lists                         | Extra class(es) to write and maintain — more boilerplate than a plain constructor                |
| Produces immutable, fully-valid objects — no partially-constructed state visible to callers | Slightly more verbose for simple objects with few fields                                         |
| Readable, self-documenting, fluent construction calls                                       | Builder itself is mutable during construction — must not be shared across threads while building |
| Centralizes validation logic in one place (`build()`)                                       | Adds indirection — an extra layer between "I want an object" and having one                      |
| Can produce different representations of a product via the same building steps              | Not needed at all in languages with named/default parameters                                     |

---

## Interview Questions

- **How is Builder different from Factory Method / Abstract Factory?** Factory patterns are about **which class** to instantiate (polymorphic creation, usually a single call). Builder is about **how** to assemble a single complex object step by step — the focus is on construction _process_, not choosing between implementations.
- **Why make the Product immutable and give the Builder a private constructor access to it?** Immutability guarantees the object can never be observed in a partially-constructed or later-mutated invalid state — once `build()` returns, the object is done and safe to share, including across threads, without defensive copying.
- **What's the role of the Director in the classic GoF Builder, and why is it often skipped today?** The Director encapsulates known _sequences_ of builder calls (standard configurations). It's often omitted in modern code because a fluent builder used directly by the client is usually clear enough on its own — the Director mainly earns its keep when you have several well-known, reusable "recipes" that are called from many places.
- **How would you enforce that certain required fields are always set before `build()` is called?** Pass required fields into the Builder's constructor (not a setter) so they can't be forgotten, and/or validate for missing/invalid required state inside `build()` and throw before constructing the Product — as shown in the POST/body example above.
- **Where have you seen Builder in real libraries?** `StringBuilder`/`StringBuffer` (though simpler, same step-accumulate-then-produce idea), Java's `Stream.Builder`, Lombok's `@Builder` annotation, `okhttp3.Request.Builder`, protobuf's generated builder classes for every message type.
- **Is Builder thread-safe?** Not inherently — a single Builder instance accumulating state should not be shared across threads mid-construction. The _Product_ it produces, once built and immutable, is safe to share freely.

---

## Related Notes

- [[factory-method]]
- [[abstract-factory]]
- [[singleton]]