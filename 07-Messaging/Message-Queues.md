# Message Queues & Async Processing

## 1. Definition

A message queue lets services communicate **asynchronously** — a producer sends a message to the queue and moves on; a consumer processes it independently, whenever it's ready. Decouples the sender's timing/availability from the receiver's.

Directly addresses [[throughput]] (absorb bursts) and [[reliability]] (don't lose work if a downstream service is briefly down).

---

## 2. Why Use a Queue (Sync vs Async)

|Synchronous (direct call)|Asynchronous (via queue)|
|---|---|
|Caller blocks waiting for response|Caller continues immediately after enqueueing|
|Both services must be up simultaneously|Consumer can be down temporarily; messages wait in the queue|
|A slow downstream service slows the caller directly|Slow consumer doesn't block the producer|
|Simple to reason about|Requires handling eventual processing, retries, ordering|

**When to reach for a queue:** the task doesn't need an immediate response (sending an email, resizing an uploaded image, processing a payment webhook, generating a report), or you need to **smooth out traffic spikes** so a downstream service isn't overwhelmed (buffer absorbs the burst, consumer drains it at a sustainable rate — ties to [[throughput]]'s batching trade-off).

---

## 3. Queue vs Pub/Sub vs Log (Core Models)

|Model|Delivery|Example systems|
|---|---|---|
|**Point-to-point queue**|Each message consumed by exactly **one** consumer|RabbitMQ, SQS|
|**Pub/Sub (fan-out)**|Each message delivered to **all** subscribers|Redis Pub/Sub, SNS, Google Pub/Sub|
|**Log/Stream**|Messages persisted in order; consumers track their own read position, can replay|Kafka, Kinesis|

**Kafka is technically a distributed log**, not a traditional queue — this distinction matters in interviews: multiple consumer groups can each independently read the **entire** stream at their own pace, and messages aren't deleted on consumption (they expire by retention policy instead).

---

## 4. Delivery Guarantees

|Guarantee|Meaning|Risk|
|---|---|---|
|**At-most-once**|Message delivered 0 or 1 times|Can lose messages (no retry)|
|**At-least-once**|Message delivered 1 or more times|Can duplicate — **consumer must be idempotent**|
|**Exactly-once**|Message delivered exactly once|Hardest to guarantee in distributed systems; usually achieved via at-least-once + idempotent consumer, or transactional producer/consumer support (Kafka has this within its own ecosystem)|

**Interview point:** "exactly-once" is often a marketing simplification — the practical, robust pattern is **at-least-once delivery + idempotent processing** (e.g. using an idempotency key to detect/skip already-processed messages), same idempotency concept discussed in [[http]] and [[reliability]].

---

## 5. Message Ordering

- **FIFO queues** (SQS FIFO, Kafka partition-level ordering) guarantee order **within a partition/queue**, not globally across a distributed system
- **Kafka**: ordering guaranteed **within a partition** only — messages with the same key always go to the same partition (similar hashing concept to [[sharding]]), preserving order for that key's events, while different keys can be processed in parallel across partitions

---

## 6. Common Patterns

### Dead Letter Queue (DLQ)

Messages that repeatedly fail processing (after N retries) are moved to a separate queue for manual inspection instead of blocking the main queue or being silently dropped. Essential for [[reliability]] — failures become visible instead of vanishing.

### Retry with Exponential Backoff

```
Attempt 1 fails → wait 1s → retry
Attempt 2 fails → wait 2s → retry
Attempt 3 fails → wait 4s → retry
... → after max attempts, send to DLQ
```

Prevents a failing consumer from hammering a struggling downstream dependency even harder.

### Fan-out

One event triggers multiple independent consumers (e.g. "order placed" → inventory service, email service, analytics service each subscribe independently) — decouples producers from needing to know all consumers, a core building block for event-driven microservices architecture (also used to fan out [[websocket]] messages across gateway servers).

### Competing Consumers

Multiple instances of the same consumer pull from one queue to parallelize processing and scale throughput horizontally — directly enables [[scalability]] for background processing workloads.

---

## 7. Kafka vs RabbitMQ vs SQS (Quick Orientation)

| |Kafka|RabbitMQ|AWS SQS|
|---|---|---|---|
|Model|Distributed log|Traditional message broker (queue + exchanges)|Managed queue service|
|Throughput|Very high (built for high-volume streaming)|Moderate|Moderate-high (managed, auto-scales)|
|Message retention|Configurable, can replay history|Deleted once consumed (typically)|Deleted once consumed (with visibility timeout)|
|Ordering|Per-partition|Per-queue (with caveats)|FIFO queues support it; standard queues don't guarantee it|
|Operational complexity|Higher (self-managed clusters, ZooKeeper/KRaft)|Moderate|Low (fully managed)|
|Best for|Event streaming, event sourcing, high-throughput pipelines, CDC (see [[replication]])|Complex routing logic, traditional task queues|Simple, managed async task queues without operational overhead|

---

## 8. Interview Talking Points

- Justify **why** async is appropriate for the specific task (don't just say "use a queue" — explain what synchronous coupling problem it solves).
- Always mention **idempotency** when discussing at-least-once delivery — a strong, correct, commonly-expected detail.
- Bring up **DLQs and retry/backoff** as the failure-handling story — shows production maturity beyond the happy path.
- For high-throughput event pipelines (analytics, activity feeds, CDC), reach for Kafka by name and explain why (replay, high throughput, multiple independent consumer groups) rather than a generic "message queue."

---

## 9. Resources

- _Designing Data-Intensive Applications_ by Martin Kleppmann — Ch. 11 (Stream Processing)
- [Confluent – Kafka Fundamentals](https://developer.confluent.io/what-is-apache-kafka/)
- [RabbitMQ – Official Tutorials](https://www.rabbitmq.com/tutorials)
- [AWS SQS Docs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)

---

## Related Notes

- [[reliability]]
- [[scalability]]
- [[throughput]]
- [[replication]]
- [[sharding]]
- [[websocket]]