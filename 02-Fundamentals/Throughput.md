# Throughput

## 1. Definition

Throughput = the **number of operations/requests a system can process per unit time**. Commonly measured as:

- **RPS / QPS** (Requests/Queries Per Second) — for web services
- **TPS** (Transactions Per Second) — for databases
- **Mbps/Gbps** — for network throughput
- **IOPS** — for storage/disk throughput

It answers: _"How much work can this system get done per second?"_ — as opposed to [[latency]], which answers "how long does one unit of work take?"

---

## 2. Throughput vs Latency — The Core Distinction

| |Latency|Throughput|
| ---|---|---|
|Measures|Time per single operation|Operations per unit time|
|Analogy|Speed of one car on a highway|Number of cars passing a point per hour|
|Improved by|Faster processing, less network hops|Parallelism, batching, more capacity|

**Important nuance:** low latency and high throughput aren't automatically both achieved together — a system can process many requests/sec (high throughput) while each individual request takes a while (high latency), if there's enough parallelism (like a highway with many lanes but a speed limit).

```
Throughput ≈ Concurrency / Latency   (Little's Law, simplified)
```

**Little's Law** (queueing theory): `L = λ × W`

- `L` = average number of requests in the system (concurrency)
- `λ` = average arrival rate (throughput)
- `W` = average time a request spends in the system (latency)

This is a genuinely useful interview tool: if you know two of these, you can derive the third — e.g. estimate how many concurrent connections a server needs to sustain a target throughput given average latency.

---

## 3. What Limits Throughput

| Bottleneck                 | Example                                                      |
| -------------------------- | ------------------------------------------------------------ |
| **CPU-bound**              | Heavy computation per request (encryption, image processing) |
| **I/O-bound**              | Disk reads/writes, network calls                             |
| **Database contention**    | Lock contention, single-writer bottlenecks                   |
| **Connection limits**      | Max concurrent connections/threads (thread pool exhaustion)  |
| **Network bandwidth**      | Raw pipe size between services/regions                       |
| **Serialization/queueing** | Single-threaded processing forcing sequential handling       |

---

## 4. Increasing Throughput — Techniques

|Technique|How it helps|
|---|---|
|**Horizontal scaling**|More instances process requests in parallel|
|**Batching**|Amortize per-operation overhead across many items (trade-off: increases per-item latency)|
|**Asynchronous / non-blocking I/O**|Don't block a thread waiting on I/O — handle more concurrent requests per node|
|**Connection pooling**|Reuse expensive connections instead of recreating|
|**Caching**|Serve repeat requests without redoing expensive work|
|**Load balancing**|Spread load evenly so no single node caps overall throughput|
|**Message queues**|Buffer bursts, let consumers process at sustainable rate (smooths throughput, doesn't necessarily raise the max)|
|**Database sharding**|Parallelize writes across shards instead of one bottlenecked primary|
|**Compression**|Reduces bytes transferred, raising effective network throughput|

---

## 5. Throughput vs Latency Trade-off (Concrete Example)

**Message batching in Kafka producers:**

- Small `batch.size`, low `linger.ms` → low latency per message, lower overall throughput (more network round-trips)
- Larger `batch.size`, higher `linger.ms` → higher throughput (fewer, bigger network calls), but each message waits longer before being sent → higher latency

This exact trade-off shows up everywhere: TCP Nagle's algorithm, database write batching, gRPC streaming, GPU batch inference.

---

## 6. Measuring & Reasoning About Throughput (Interview Framework)

Back-of-envelope example: _"Design a system for 10M daily active users, each making 20 requests/day."_

```
Total requests/day = 10M × 20 = 200M
Average RPS = 200M / 86,400 sec ≈ 2,315 RPS
Peak RPS (assume 3x average for peak hours) ≈ 6,945 RPS
```

Always calculate **average AND peak** — peak is what you design capacity for.

---

## 7. Interview Talking Points

- Clarify with the interviewer whether the priority is **latency-sensitive** (e.g. live chat, gaming) or **throughput-sensitive** (e.g. batch analytics, log ingestion) — design decisions differ significantly.
- Bring up **Little's Law** when asked to estimate required server/connection capacity — it signals quantitative rigor.
- Mention that increasing throughput via horizontal scaling has diminishing returns if there's a shared bottleneck downstream (e.g. all instances hitting one DB) — tie back to [[scalability]].

---

## 8. Resources

- [Little's Law – explained (Baeldung)](https://www.baeldung.com/cs/littles-law)
- _Designing Data-Intensive Applications_ by Martin Kleppmann — Ch. 1 (Throughput vs response time)
- [Kafka Docs – Producer batching/linger.ms](https://kafka.apache.org/documentation/#producerconfigs_linger.ms)
- [System Design Primer – Performance vs Scalability](https://github.com/donnemartin/system-design-primer#performance-vs-scalability)

---

## Related Notes

- [[latency]]
- [[scalability]]
- [[availability]]
- [[system-design-networking-index]]