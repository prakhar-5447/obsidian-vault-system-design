# RabbitMQ

## 1. Definition

RabbitMQ is a **traditional message broker** implementing the AMQP (Advanced Message Queuing Protocol) model: producers publish messages to **exchanges**, exchanges route messages to **queues** based on rules/bindings, and consumers pull (or receive pushed) messages from queues. Unlike [[kafka]]'s log model, messages are **removed once acknowledged** by a consumer. Strong fit for task queues, work distribution, and flexible routing — a core piece of many [[event-driven-architecture]] designs.
## 2. Core Concepts

|Term|Meaning|
|---|---|
|**Producer**|Publishes messages to an exchange (never directly to a queue)|
|**Exchange**|Routes incoming messages to one or more queues based on routing rules|
|**Queue**|Stores messages until a consumer processes them; FIFO within a queue|
|**Binding**|Rule connecting an exchange to a queue (with an optional routing key)|
|**Consumer**|Subscribes to a queue and receives messages (broker pushes to consumer)|
|**Acknowledgment (ack)**|Consumer confirms successful processing; unacked messages are redelivered|

---

## 3. Exchange Types (Routing Flexibility)

|Exchange Type|Routing Behavior|Use Case|
|---|---|---|
|**Direct**|Routes to queue(s) whose binding key exactly matches the message's routing key|Point-to-point task routing (e.g. route by task type)|
|**Fanout**|Broadcasts to **all** bound queues, ignores routing key|Pub-sub style broadcast to multiple consumers (see [[pub-sub]])|
|**Topic**|Routes based on wildcard pattern matching on routing key (`*` = one word, `#` = zero or more words)|Flexible multi-criteria routing (e.g. `orders.us.*`, `logs.#.error`)|
|**Headers**|Routes based on message header attributes instead of routing key|Routing on structured metadata rather than a string key|

This routing flexibility is RabbitMQ's key differentiator from Kafka's simpler topic-only model.

---

## 4. Delivery Guarantees

- **Consumer acknowledgments** — a message stays in the queue (or is redelivered) until the consumer explicitly acks it; if the consumer crashes before acking, the message is requeued → **at-least-once delivery** by default
- **Publisher confirms** — broker confirms receipt back to the producer, so the producer knows the message was safely queued (protects against producer-side loss)
- **Durable queues + persistent messages** — messages survive broker restarts when both the queue is declared durable and messages are marked persistent (adds write latency)
- **Dead Letter Exchanges (DLX)** — messages that are rejected, expire (TTL), or exceed retry limits get routed to a separate exchange/queue instead of being silently dropped — critical for debugging and manual reprocessing

---

## 5. Work Queues & Load Balancing

- Multiple consumers on the same queue get messages **round-robin** by default (competing consumers pattern) — a natural way to horizontally scale workers
- **Prefetch count** limits how many unacked messages a consumer can hold at once, preventing one fast producer from overwhelming a slow consumer's memory
- Good fit for classic task-queue workloads: background jobs, email sending, image processing, RPC-style request/reply patterns (RabbitMQ has native RPC support via reply-to queues)

---

## 6. Clustering & High Availability

- RabbitMQ clusters replicate queues across nodes for HA (via **quorum queues**, the modern replacement for the older mirrored queues)
- Quorum queues use the Raft consensus protocol for leader election and replication, trading some throughput for stronger consistency/durability guarantees
- Unlike Kafka's partition-per-broker log model, RabbitMQ queues aren't inherently partitioned — very high throughput single-queue workloads scale less naturally than Kafka's partitioned topics

---

## 7. RabbitMQ vs Kafka — When to Pick Which

|Need|Pick|
|---|---|
|Complex routing (multiple criteria, wildcards, per-message decisions)|**RabbitMQ**|
|Classic background job/task queue with worker pool|**RabbitMQ**|
|Request/reply (RPC) messaging|**RabbitMQ**|
|Very high throughput event streaming / log aggregation|**[[kafka]]**|
|Need to replay history / event sourcing|**[[kafka]]**|
|Many independent consumer groups reading the same stream|**[[kafka]]**|
|Strict message ordering per key, at massive scale|**[[kafka]]** (per-partition)|

See [[kafka]] for the full head-to-head comparison table.

---

## 8. Interview Talking Points

- RabbitMQ is the answer when the question emphasizes **routing complexity** or **task distribution among workers**, not raw throughput/streaming
- Be explicit about the **exchange → binding → queue** model — interviewers often want to see you understand routing isn't just "topic in, topic out" like Kafka
- Mention **acks + DLX** together as the durability/error-handling story
- If asked about scaling to Kafka-like volumes, be honest that RabbitMQ isn't the natural fit — that's a signal you understand the trade-off space, not a weakness to hide

---

## 9. Resources

- [RabbitMQ Official Docs – Tutorials](https://www.rabbitmq.com/tutorials)
- [RabbitMQ – AMQP Concepts](https://www.rabbitmq.com/tutorials/amqp-concepts.html)
- [CloudAMQP – RabbitMQ vs Kafka](https://www.cloudamqp.com/blog/rabbitmq-vs-kafka.html)

---

## Related Notes

- [[kafka]]
- [[event-driven-architecture]]
- [[pub-sub]]
- [[message-queues]]
- [[rate-limiting]]
- [[reliability]]