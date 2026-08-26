# Apache Kafka

## 1. Definition

Kafka is a **distributed, append-only log** used as a high-throughput messaging/streaming platform. Producers append records to **topics**; consumers read from them independently at their own pace. Unlike a traditional queue, messages **aren't removed on consumption** — they're retained for a configurable period, so multiple consumers can replay the same log independently. Core building block for [[event-driven-architecture]] and stream processing.

---

## 2. Core Concepts

|Term|Meaning|
|---|---|
|**Topic**|Named stream of records (like a table or category)|
|**Partition**|A topic is split into ordered, immutable partitions — the unit of parallelism|
|**Offset**|Each record's position within a partition; consumers track their own offset|
|**Producer**|Writes records to a topic (optionally choosing a partition key)|
|**Consumer**|Reads records from a topic starting at some offset|
|**Consumer Group**|Set of consumers sharing the work of a topic — each partition is read by exactly one consumer within a group|
|**Broker**|A single Kafka server; a cluster is many brokers|
|**ZooKeeper / KRaft**|Cluster metadata/coordination — legacy ZooKeeper is being replaced by Kafka's own **KRaft** consensus protocol|

---

## 3. Partitions — Kafka's Core Scaling Mechanism

- A topic is split into N partitions, each an ordered log
- **Ordering is only guaranteed within a partition**, not across the whole topic
- Producers typically hash a **partition key** (e.g. `userId`) so all records for the same key land in the same partition — this is how you get per-key ordering
- More partitions = more parallelism (more consumers can read concurrently), but also more overhead and more complexity for global ordering

```
Topic: orders (3 partitions)
  Partition 0: [msg0, msg3, msg6, ...]
  Partition 1: [msg1, msg4, msg7, ...]
  Partition 2: [msg2, msg5, msg8, ...]
```

---

## 4. Consumer Groups & Scaling Reads

- Each partition is consumed by **exactly one** consumer within a given group at a time — this is how Kafka parallelizes consumption without duplicate processing inside a group
- If consumers > partitions, extra consumers sit idle
- If consumers < partitions, some consumers handle multiple partitions
- **Multiple consumer groups** can independently read the _same_ topic from the start — this is what enables fan-out (e.g. analytics and billing both consuming `OrderPlaced` independently)
- **Rebalancing** happens when consumers join/leave a group — partitions get reassigned, briefly pausing consumption

---

## 5. Replication & Durability

- Each partition is replicated across multiple brokers (**replication factor**, commonly 3)
- One replica is the **leader** (handles all reads/writes); others are **followers** that replicate the log
- **In-Sync Replicas (ISR)** — followers caught up with the leader; only ISR members are eligible for leader election on failure
- **acks** setting controls durability vs latency trade-off:

|`acks`|Meaning|Trade-off|
|---|---|---|
|`0`|Producer doesn't wait for any ack|Fastest, can lose data|
|`1`|Leader acks after writing locally|Faster, can lose data if leader fails before replicating|
|`all` / `-1`|Leader waits for all ISR to replicate|Slowest, strongest durability guarantee|

---

## 6. Delivery Semantics & Ordering

- Kafka natively supports **at-least-once** delivery by default
- **Exactly-once semantics (EOS)** are achievable via idempotent producers + transactional writes (Kafka Transactions API), but add complexity/overhead — often "effectively-once" in practice
- Consumers commit offsets after processing; committing too early risks message loss on crash, committing too late risks reprocessing (duplicates) — same idempotency discussion as in [[event-driven-architecture]]

---

## 7. Retention & Replay

- Records are retained for a configured time/size (not deleted on read) — e.g. 7 days by default
- **Log compaction** is an alternative retention mode: keeps only the latest record per key (good for representing current state, e.g. a changelog topic)
- Because records aren't deleted on consumption, a new consumer group can **replay the entire topic from offset 0** — this is what makes Kafka useful for reprocessing, backfills, and event sourcing, unlike a traditional queue

---

## 8. Kafka vs Traditional Message Queue (RabbitMQ)

| |Kafka|[[rabbitmq]]|
|---|---|---|
|Model|Distributed log, pull-based|Traditional broker, push-based|
|Message retention|Retained after consumption (replay-able)|Deleted once acknowledged/consumed|
|Ordering|Per-partition|Per-queue (FIFO)|
|Throughput|Very high (built for streaming/logs)|High, but generally lower than Kafka|
|Routing|Simple (topic-based)|Rich (exchanges: direct/topic/fanout/headers)|
|Best for|Event streaming, log aggregation, event sourcing, high-volume pipelines|Task queues, complex routing, RPC-style workloads|

See [[rabbitmq]] for the queue-model comparison in depth.

---

## 9. Interview Talking Points

- Kafka is the go-to answer for **high-throughput event streaming**, **log aggregation**, and **event sourcing** — not for complex per-message routing (that's RabbitMQ's strength)
- Be ready to explain: partitions → parallelism, replication → durability, consumer groups → independent scaling of consumption, retention → replay
- Name the ordering caveat explicitly: **global ordering is not guaranteed**, only per-partition — a classic follow-up question
- Bring up `acks` and ISR when asked about durability trade-offs
- Mention KRaft as the current direction away from ZooKeeper dependency

---

## 10. Resources

- [Kafka Official Docs](https://kafka.apache.org/documentation/)
- [Confluent – Kafka Architecture](https://developer.confluent.io/courses/architecture/get-started/)
- [Jay Kreps – The Log: What every software engineer should know](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)

---

## Related Notes

- [[event-driven-architecture]]
- [[rabbitmq]]
- [[pub-sub]]
- [[message-queues]]
- [[rate-limiting]]
- [[reliability]]