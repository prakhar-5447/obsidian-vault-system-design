# Publish-Subscribe (Pub-Sub)

## 1. Definition

Pub-Sub is a messaging pattern where **publishers** send messages to a **topic/channel** without knowing who's listening, and **subscribers** register interest in a topic to receive all messages published to it. It's a _pattern_, not a specific technology — [[kafka]] topics, [[rabbitmq]] fanout exchanges, Redis Pub/Sub, and managed services like Google Cloud Pub/Sub, AWS SNS, and Azure Service Bus Topics are all implementations of it. Foundational pattern underlying most [[event-driven-architecture]].

---

## 2. Core Concepts

|Term|Meaning|
|---|---|
|**Publisher**|Sends messages to a topic, unaware of subscriber count or identity|
|**Subscriber**|Registers interest in a topic; receives a copy of every message published to it|
|**Topic / Channel**|Named destination messages are published to|
|**Fan-out**|One message delivered to **all** subscribers (as opposed to queue semantics where one message goes to exactly one consumer)|

**Key distinction from a work queue:** in a queue, each message is consumed by exactly one worker (competing consumers, for load distribution). In pub-sub, each message is delivered to **every** subscriber (broadcast, for independent parallel reactions). Kafka blurs this: within a consumer group it behaves like a queue (each partition to one consumer); across consumer groups it behaves like pub-sub (every group gets its own copy).

---

## 3. Delivery Models

### Push-based

Broker actively delivers messages to subscribers as they arrive (e.g. webhooks, Redis Pub/Sub, AWS SNS → HTTP endpoint). Lower latency, but subscriber must be available and can be overwhelmed by bursts.

### Pull-based

Subscriber polls/reads from the broker at its own pace (e.g. Kafka consumers, Google Cloud Pub/Sub pull subscriptions). Subscriber controls its own load, easier to scale and retry, slightly higher latency.

---

## 4. Delivery Guarantees & At-Least-Once Reality

- Most pub-sub systems default to **at-least-once** delivery — a message may be delivered more than once (e.g. if an ack is lost or a subscriber restarts before acking)
- **Redis Pub/Sub is a notable exception**: it's **fire-and-forget** with no persistence — if a subscriber isn't connected when a message is published, it's lost. Fine for ephemeral notifications (e.g. live chat presence), wrong choice for anything requiring durability
- Durable pub-sub systems (Kafka topics, Cloud Pub/Sub, SNS+SQS) persist messages until acknowledged/retained, so subscribers that reconnect can still catch up
- As with all at-least-once systems, **subscribers must be idempotent** — see [[event-driven-architecture]] for the general pattern

---

## 5. Common Architecture: Pub-Sub + Queue (Fan-out to Workers)

A very common production pattern combines both models: a pub-sub layer fans a message out to multiple **queues**, each queue feeding a pool of workers that load-balance the work for that subscriber.

```
Publisher → Topic (SNS) → fan-out to →  Queue A (SQS) → Worker pool A
                                    →  Queue B (SQS) → Worker pool B
                                    →  Queue C (SQS) → Worker pool C
```

This gives you the best of both: every subscribing system gets its own copy of the event (pub-sub), and within each subscribing system, work is load-balanced across a worker pool (queue). This exact pattern is why AWS pairs **SNS (pub-sub) + SQS (queue)** so often.

---

## 6. When to Reach for Pub-Sub

- Multiple independent systems need to react to the same event (e.g. `UserSignedUp` triggers welcome email, analytics, CRM sync, fraud check — each owned by a different team)
- You want to **decouple** the producer from knowing anything about consumers, including how many there are
- You need to add a new consumer of an existing event stream **without touching the publisher** — a major architectural win for evolving systems over time

Not the right tool when you need exactly one worker to process each message (use a plain queue) or when you need complex per-message routing logic (use [[rabbitmq]] exchanges).

---

## 7. Interview Talking Points

- Clarify pub-sub vs queue semantics explicitly if the interviewer's question is ambiguous — "do you mean each message goes to one consumer, or every consumer?" is a good clarifying question that shows you know the distinction
- Bring up the **SNS+SQS fan-out pattern** as a concrete real-world architecture when asked how to notify multiple downstream systems reliably
- Mention Redis Pub/Sub's lack of durability as a cautionary example — a good way to show you think about failure modes, not just the happy path
- Tie back to idempotency and at-least-once delivery — same story as [[event-driven-architecture]] and [[kafka]]

---

## 8. Resources

- [AWS – SNS + SQS Fan-out Pattern](https://docs.aws.amazon.com/sns/latest/dg/sns-common-scenarios.html)
- [Google Cloud – Pub/Sub Documentation](https://cloud.google.com/pubsub/docs/overview)
- [Redis Docs – Pub/Sub](https://redis.io/docs/latest/develop/interact/pubsub/)

---

## Related Notes

- [[event-driven-architecture]]
- [[kafka]]
- [[rabbitmq]]
- [[message-queues]]
- [[rate-limiting]]
- [[reliability]]