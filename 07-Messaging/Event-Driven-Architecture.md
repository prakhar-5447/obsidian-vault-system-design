# Event-Driven Architecture (EDA)

## 1. Definition

Event-driven architecture is a design pattern where services communicate by **producing and consuming events** (facts that something happened) rather than calling each other directly. Producers emit events without knowing who (if anyone) consumes them; consumers react asynchronously. Usually built on a [[message-queues|message broker]] like [[kafka]], [[rabbitmq]], or a [[pub-sub]] system.

Contrast with **request-response** (synchronous, caller waits, tight coupling) — EDA decouples producer and consumer in time, space, and synchronization.

---

## 2. Why Event-Driven

- **Decoupling** — producers and consumers don't need to know about each other, only the event schema/topic
- **Scalability** — consumers scale independently; slow consumers don't block producers
- **Resilience** — if a consumer is down, events queue up instead of failing requests
- **Multiple consumers** — one event can trigger many independent workflows (e.g. `OrderPlaced` → billing, shipping, analytics, notifications) without the producer knowing about any of them
- **Natural audit trail** — the event stream itself is a log of everything that happened

---

## 3. Core Concepts

|Term|Meaning|
|---|---|
|**Event**|An immutable fact that something happened (`OrderPlaced`, `UserSignedUp`) — past tense, not a command|
|**Producer / Publisher**|Emits events, doesn't know or care who consumes them|
|**Consumer / Subscriber**|Reacts to events; may itself produce new events (event chaining)|
|**Topic / Channel**|Named stream events are published to|
|**Broker**|Middleware that routes events from producers to consumers ([[kafka]], [[rabbitmq]])|
|**Event Bus**|Logical abstraction over the broker that routes events to interested parties|

**Event vs Command:** an event says "this happened" (fact, broadcast, no expected reply); a command says "do this" (imperative, usually directed at one specific handler). Rate-limiting or API design tends to be command/request-driven; EDA is event-driven.

---

## 4. Common Patterns

### Event Notification

Producer emits a lightweight event (`OrderPlaced: {orderId: 123}`). Consumers that need more data call back to the source service. Keeps events small but reintroduces coupling via follow-up calls.

### Event-Carried State Transfer

The event carries the full data consumers need (`OrderPlaced: {orderId, items, total, address}`). Consumers don't need to call back — fully decoupled — but events are larger and schema changes are more sensitive.

### Event Sourcing

Instead of storing current state, store the **sequence of events** that led to it; current state is derived by replaying events. Gives a full audit log and the ability to reconstruct state at any point in time, at the cost of query complexity (need projections/read models).

### CQRS (Command Query Responsibility Segregation)

Split the write model (handles commands, often event-sourced) from the read model (optimized, denormalized projections built by consuming events). Frequently paired with event sourcing but not required.

### Choreography vs Orchestration

| |Choreography|Orchestration|
|---|---|---|
|How it works|Each service reacts to events and emits its own — no central coordinator|A central orchestrator explicitly calls each step and tracks state|
|Coupling|Low — services only know event schemas|Higher — orchestrator knows the whole workflow|
|Visibility|Harder to see the overall flow (implicit, spread across services)|Easy to see the whole saga in one place|
|Failure handling|Each service handles its own compensation|Orchestrator manages compensation/rollback centrally|
|Common use|Simple, few-step flows|Complex multi-step business transactions (**Saga pattern**)|

---

## 5. Delivery Semantics

|Guarantee|Meaning|Trade-off|
|---|---|---|
|**At-most-once**|Event delivered 0 or 1 times|Fast, simplest, but can silently lose events|
|**At-least-once**|Event delivered ≥1 times, may duplicate|Most common default — requires **idempotent consumers**|
|**Exactly-once**|Event delivered and processed exactly once|Hardest to guarantee; usually "effectively-once" via dedup + idempotency, not true exactly-once across a distributed system|

**Idempotency is the practical answer**: design consumers so processing the same event twice has no additional effect (e.g. dedupe by event ID, upsert instead of insert, check-then-act with a unique constraint).

---

## 6. Challenges

- **Eventual consistency** — consumers see events after the fact, not instantly; system-wide state may be briefly inconsistent
- **Debugging/tracing** — a single business flow is spread across many async handlers; needs correlation IDs and distributed tracing
- **Event schema evolution** — producers and consumers deploy independently, so schemas need backward/forward compatibility (versioning, schema registries)
- **Ordering** — many brokers only guarantee order within a partition/queue, not globally (see [[kafka]] partitioning)
- **Duplicate/out-of-order delivery** — see [[idempotency]] above
- **Testing complexity** — integration tests need to simulate async timing, not just call-and-assert

---

## 7. Where It Fits in a System Design Interview

- Reach for EDA when asked about **decoupling services**, **fan-out to multiple consumers**, **long-running workflows**, or **audit/replay requirements**
- Name the broker choice and justify it: [[kafka]] for high-throughput ordered logs and replay; [[rabbitmq]] for flexible routing and task queues; managed [[pub-sub]] for simple fan-out without operating infrastructure
- Bring up delivery semantics and idempotency unprompted — interviewers listen for this
- Mention the Saga pattern for distributed transactions across services (since 2PC doesn't scale well in microservices)

---

## 8. Resources

- [Martin Fowler – What do you mean by Event-Driven?](https://martinfowler.com/articles/201701-event-driven.html)
- [Confluent – Event-Driven Architecture Overview](https://www.confluent.io/learn/event-driven-architecture/)
- [Microsoft – Event-Driven Architecture Style](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven)
- [AWS – Saga Pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-pattern.html)

---

## Related Notes

- [[kafka]]
- [[rabbitmq]]
- [[pub-sub]]
- [[message-queues]]
- [[rate-limiting]]
- [[reliability]]
- [[api-gateway]]