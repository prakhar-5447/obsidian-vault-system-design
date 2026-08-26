UML

UML is a standardized visual notation for describing the **structure** and **behavior** of a software system, independent of any specific programming language. In an LLD interview, you rarely need full UML rigor — a clear, readable approximation (as used throughout this vault's Class Diagram / Sequence Flow sections) communicates the same ideas without needing exact notation memorized. Still worth knowing the standard symbols, since interviewers and whiteboards do use them.

---

## Class Diagram

**Purpose:** shows the **static structure** of a system — classes, their attributes/methods, and the relationships between them. This is the diagram type used throughout this vault's "Class Diagram" sections.

### Notation

```text
┌─────────────────────────┐
│        ClassName            │
├─────────────────────────┤
│ - privateField: Type          │   ("-" = private, "+" = public, "#" = protected)
│ + publicField: Type            │
├─────────────────────────┤
│ + publicMethod(param: Type)      │
│ - privateMethod(): ReturnType      │
└─────────────────────────┘
```

### Relationship Types

|Relationship|Notation|Meaning|Example|
|---|---|---|---|
|**Association**|`A ──── B` (plain line)|A general "uses/knows about" link between two classes|`Teacher ──── Student`|
|**Aggregation**|`A ◇──── B` (hollow diamond at A)|**Has-a**, but B can exist independently of A ("weak" ownership)|`Department ◇──── Professor` (professor exists without the department)|
|**Composition**|`A ◆──── B` (filled diamond at A)|**Has-a**, and B's lifecycle is bound to A ("strong" ownership — B is destroyed when A is)|`House ◆──── Room` (a room doesn't exist without its house)|
|**Inheritance / Generalization**|`A ──▷ B` (hollow triangle pointing at parent)|**Is-a** — A extends/inherits from B|`Dog ──▷ Animal`|
|**Realization / Implementation**|`A┄┄▷ B` (dashed line, hollow triangle)|A implements interface B|`StripeAdapter ┄┄▷ PaymentProcessor`|
|**Dependency**|`A┄┄▷ B` (dashed line, open arrow)|A depends on B temporarily (e.g. a method parameter), not a stored field|A method that takes `B` as a parameter but doesn't hold onto it|

Aggregation vs Composition is the distinction interviewers probe most — "if the container is destroyed, does the contained object still make sense to exist?" Yes → aggregation. No → composition.

### Example

```text
┌────────────────────┐
│      Library          │
├────────────────────┤
│ - books: List<Book>      │
├────────────────────┤
│ + addBook(book: Book)      │
│ + findBook(title: String)    │
└──────────┬─────────┘
           │ ◆ composition (Book doesn't exist
           │   independently of the Library catalog)
           ▼
┌────────────────────┐        ┌────────────────────┐
│        Book            │◀───────│  <<interface>>        │
│ - title: String           │        │     Borrowable          │
│ - author: String            │        │  + borrow()                │
└────────────────────┘        │  + returnItem()              │
           ▲                      └────────────────────┘
           │ ┄┄▷ realization
┌────────────────────┐
│    EBook               │
│  (implements            │
│   Borrowable)             │
└────────────────────┘
```

---

## Sequence Diagram

**Purpose:** shows **behavior over time** — the order of messages/calls exchanged between objects for a specific scenario/use case. This is the diagram type used throughout this vault's "Sequence Flow" sections (usually simplified to arrow-and-indent text rather than full lifeline notation).

### Notation

```text
Actor          ObjectA          ObjectB
  │                │                │
  │──── request ──▶│                │
  │                │──── call() ───▶│
  │                │                │──┐
  │                │                │  │ (self-call / internal processing)
  │                │                │◀─┘
  │                │◀─── result ────│
  │◀─── response ──│                │
  │                │                │
```

- **Vertical lines (lifelines)** — represent each object's existence over the timeline (time flows top to bottom)
- **Solid arrows** — synchronous method calls (caller waits for a response)
- **Dashed arrows** — return values/responses
- **Open arrowheads** — asynchronous calls (caller doesn't wait — relevant when depicting [[event-driven-architecture]] or message queue interactions)
- **Activation bars** (thin rectangles on a lifeline) — show when an object is actively executing/processing
- **Alt/opt fragments** (boxed regions labeled `alt`/`opt`/`loop`) — represent conditional branches or loops within the sequence

### Example

```text
Customer        VendingMachine       Inventory
  │                   │                   │
  │─ insertMoney(100)▶│                   │
  │                   │  (Idle → HasMoney)│
  │                   │                   │
  │─ selectProduct(A5)▶                   │
  │                   │─ isAvailable(A5)─▶│
  │                   │◀──── true ────────│
  │                   │  (validate funds) │
  │                   │  (HasMoney → ProductSelected)
  │                   │                   │
  │─── dispense() ────▶                   │
  │                   │─ reduceStock(A5) ▶│
  │                   │  (ProductSelected → Idle)
  │◀── product+change─│                   │
```

Sequence diagrams are the natural companion to a Class Diagram: the class diagram tells you _what exists_, the sequence diagram tells you _how those things collaborate for one specific scenario_.

---

## Activity Diagram

**Purpose:** models **workflow/control flow** — the logical steps, decisions, and parallel paths of a process, closer to a flowchart than an object-collaboration diagram. Useful when the thing you're describing is a **process or algorithm** (e.g. an order-fulfillment workflow) rather than which objects talk to which.

### Notation

```text
        ●  (start node — filled circle)
        │
        ▼
  ┌──────────┐
  │  Action    │  (rounded rectangle — a single step/activity)
  └──────────┘
        │
        ▼
       ◇             (diamond — decision point)
      ╱ ╲
  yes╱     ╲no
    ▼         ▼
┌──────┐  ┌──────┐
│StepA   │  │StepB   │
└──────┘  └──────┘
     ╲       ╱
       ╲   ╱
         ▼
        ●  (end node — filled circle with ring)

  ═══  (fork/join bar — split into parallel activities, then rejoin)
```

- **Rounded rectangle** — an action/activity step
- **Diamond** — a decision (branches based on a condition)
- **Filled circle** — start
- **Filled circle with ring** — end
- **Thick horizontal/vertical bar** — fork (split into concurrent flows) or join (wait for concurrent flows to finish)
- **Swimlanes** (vertical/horizontal partitions) — optionally show which actor/component is responsible for each activity

### Example

```text
        ●
        │
        ▼
┌────────────────┐
│  Receive Order    │
└────────────────┘
        │
        ▼
       ◇ In stock?
      ╱ ╲
   yes╱     ╲no
     ▼         ▼
┌──────────┐ ┌────────────────┐
│Charge card │ │Notify customer   │
└──────────┘ │  out of stock       │
     │          └────────────────┘
     ▼                   │
┌──────────┐             ▼
│Ship order  │            ●
└──────────┘
     │
     ▼
     ●
```

---

## When to Use Each

|Diagram|Answers the question|Use it when|
|---|---|---|
|**Class Diagram**|"What are the pieces, and how are they structurally related?"|Designing the object model — classes, fields, methods, inheritance/composition relationships. This is what you draw first in most LLD interviews, and what this vault's "Class Diagram" sections capture|
|**Sequence Diagram**|"In this specific scenario, what calls happen, in what order, between which objects?"|Explaining a specific use case/flow end-to-end (e.g. "walk me through what happens when a user places an order") — this is what this vault's "Sequence Flow" sections capture|
|**Activity Diagram**|"What's the logical process/algorithm, including branches and parallel steps?"|Describing a workflow or business process with decisions and possibly concurrent steps, independent of which specific objects are involved — useful for onboarding flows, approval pipelines, or algorithmic logic where "which class calls which" is secondary to "what's the actual logic"|

**Practical interview guidance:**

- Start with a **Class Diagram** to nail down the object model and relationships — this is almost always the first thing an interviewer wants to see, and it's the artifact most LLD problems in this vault lead with
- Follow with a **Sequence Diagram** for the 1–2 most important/non-trivial use cases, to prove the class model actually supports the required behavior end-to-end (not just that the classes _look_ right)
- Reach for an **Activity Diagram** only when the interviewer specifically wants to see decision logic or process flow independent of object collaboration (rarer in most LLD interviews, more common in business-process/workflow-heavy problems)
- You don't need textbook-exact UML notation in a whiteboard/interview setting — clear boxes, arrows, and labels that unambiguously convey structure and flow (as used throughout this vault) are what's actually being evaluated, not notation trivia

---

## Related Notes

- [[oops]]
- [[solid]]
- [[tic-tac-toe]]
- [[splitwise]]
- [[vending-machine]]