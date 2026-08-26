# Scalability

## 1. Definition

Scalability = a system's ability to **handle increasing load** (more users, data, or traffic) by adding resources — **without a redesign** and ideally with proportional (or better) performance.

It answers: _"What happens to this system when load 10x's? Can we handle it, and how?"_

---

## 2. Vertical vs Horizontal Scaling

| |Vertical Scaling (Scale Up)|Horizontal Scaling (Scale Out)|
|---|---|---|
|**How**|Add more resources (CPU/RAM/disk) to one machine|Add more machines|
|**Ceiling**|Hardware limit — eventually you can't add more|Practically unlimited (add more nodes)|
|**Complexity**|Simple — no code changes usually needed|Complex — needs load balancing, data partitioning, coordination|
|**Downtime**|Often requires downtime to resize|Can add nodes with zero downtime|
|**Cost curve**|High-end hardware is disproportionately expensive|Commodity hardware, more linear cost|
|**Single point of failure**|Yes — one machine|No — inherently redundant|

**Interview default:** for large systems, horizontal scaling is the standard answer, but always mention vertical scaling as the simpler first step for moderate load.

---

## 3. What Makes a System Hard to Scale Horizontally

- **Statefulness** — servers holding session/local state can't be freely load-balanced (fix: externalize state to Redis/DB, or sticky sessions)
- **Single-writer bottlenecks** — one DB primary handling all writes (fix: sharding, or CQRS with separate write/read paths)
- **Shared mutable state** — locks/contention across nodes limit parallelism
- **Chatty inter-service calls** — N+1 network calls amplify latency at scale (see [[latency]])

A **stateless** service (no local session/data dependency) is trivially horizontally scalable — this is why "keep services stateless" is a core system design principle.

---

## 4. Key Scaling Techniques

|Technique|Purpose|
|---|---|
|**Load balancing**|Distribute requests across multiple instances|
|**Caching** (CDN, Redis, in-memory)|Reduce repeated work / DB load|
|**Database replication**|Read replicas scale read throughput|
|**Database sharding/partitioning**|Split data across nodes to scale writes & storage|
|**Asynchronous processing / message queues**|Decouple slow work from the request path (Kafka, SQS, RabbitMQ)|
|**Microservices**|Scale individual components independently based on their own load|
|**Auto-scaling**|Dynamically add/remove instances based on load metrics|
|**Content Delivery Networks (CDN)**|Offload static content delivery from origin servers|
|**Rate limiting / backpressure**|Protect system from overload rather than collapsing|

---

## 5. Database Scaling — Sharding Strategies

|Strategy|How it works|Trade-off|
|---|---|---|
|**Range-based sharding**|Split by key range (e.g. user_id 1-1000, 1001-2000)|Simple, but risk of hot shards|
|**Hash-based sharding**|Hash(key) determines shard|Even distribution, but range queries become hard|
|**Directory-based sharding**|Lookup service maps key → shard|Flexible, but adds a lookup hop + SPOF risk|
|**Geo-based sharding**|Shard by user region|Good for latency & compliance (data residency)|

---

## 6. Scalability vs Performance (Common Confusion)

- **Performance** = how fast the system responds _under current load_.
- **Scalability** = how well performance holds up _as load increases_.

A system can be fast at low load (good performance) but scale poorly (degrades sharply as load grows) — always evaluate both.

---

## 7. Scalability Bottleneck Identification (Interview Framework)

When asked to "scale this system," walk through:

1. **Identify the bottleneck** — is it CPU, DB reads, DB writes, network, or a single service?
2. **Read-heavy?** → caching, read replicas, CDN
3. **Write-heavy?** → sharding, async processing/queues, batching writes
4. **Compute-heavy?** → horizontal scaling of stateless workers, autoscaling
5. **Fan-out heavy (many small calls)?** → batching, denormalization, precomputation

---

## 8. Interview Talking Points

- Always state assumptions about scale explicitly (e.g. "assuming 1M DAU, ~1000 req/sec peak") before designing — interviewers want to see back-of-envelope estimation.
- Mention that scaling introduces new problems: consistency challenges ([[consistency]], [[cap-theorem]]), operational complexity, and cost — scaling isn't "free," it's a trade-off.
- Distinguish scaling **reads** (usually easier — caching, replicas) from scaling **writes** (harder — requires sharding/partitioning).

---

## 9. Resources

- _Designing Data-Intensive Applications_ by Martin Kleppmann — Ch. 1 & 6 (Partitioning)
- [High Scalability blog](http://highscalability.com/) — real-world case studies (how Twitter/Instagram/Discord scaled)
- [System Design Primer (GitHub)](https://github.com/donnemartin/system-design-primer) — comprehensive free reference
- [AWS – Scalability best practices](https://docs.aws.amazon.com/wellarchitected/latest/framework/rel-scale.html)

---

## Related Notes

- [[throughput]]
- [[latency]]
- [[availability]]
- [[consistency]]
- [[system-design-networking-index]]