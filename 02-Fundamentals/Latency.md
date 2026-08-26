# Latency

## 1. Definition

Latency = the **time delay** between a request being sent and a response being received. Usually measured in milliseconds (ms).

It answers: _"How long does one operation take?"_ (as opposed to [[throughput]], which answers "how many operations per unit time?")

---

## 2. Where Latency Comes From (Breakdown)

For a typical web request:

```
Client → DNS lookup → TCP handshake → TLS handshake → Request sent
       → Server processing → DB query → Response sent → Client renders
```

|Source|Typical contribution|
|---|---|
|DNS resolution|0 (cached) – 100s of ms (cold)|
|TCP handshake|~1 RTT|
|TLS handshake|~1–2 RTT (TLS 1.3 reduced this)|
|Network propagation|Depends on physical distance (~5ms per 1000km, roughly, given speed of light in fiber)|
|Server processing|App logic, serialization|
|Database query|Disk I/O, lock contention, query complexity|
|Queueing delay|Time spent waiting behind other requests|

**Interview tip:** always separate **network latency** (physics — can't be fully eliminated, only reduced via CDNs/edge/anycast) from **processing latency** (engineering — can be reduced via caching, indexing, async processing).

---

## 3. Latency Numbers Every Engineer Should Know (Jeff Dean's classic reference, approximate)

|Operation|Latency|
|---|---|
|L1 cache reference|~1 ns|
|Main memory reference|~100 ns|
|SSD random read|~16 μs (0.016 ms)|
|Read 1MB sequentially from memory|~3 μs|
|Round trip within same datacenter|~0.5 ms|
|Read 1MB sequentially from SSD|~50 μs|
|Disk seek (HDD)|~2-10 ms|
|Read 1MB sequentially from network|~10 ms|
|Round trip CA→Netherlands (intercontinental)|~150 ms|

**Takeaway for interviews:** a single cross-region network round trip (~150ms) can be 100,000x+ slower than a memory access. This is the single biggest argument for **caching close to the user (CDN/edge)** and **avoiding unnecessary network hops** (e.g. N+1 query problems).

---

## 4. Latency Percentiles — p50, p95, p99, p999

**Never use average latency alone** — it hides tail latency, which matters a lot for user experience at scale.

|Percentile|Meaning|
|---|---|
|p50 (median)|Half of requests are faster than this|
|p95|95% of requests are faster; 5% are slower ("the bad ones")|
|p99|99% of requests are faster; the 1% "tail"|
|p999|Even rarer, but often what power users / high-traffic paths hit constantly|

**Why tail latency matters:** if a page makes 100 backend calls and each has a 1% chance of hitting the p99 slow path, the probability that _at least one_ call is slow is `1 - 0.99^100 ≈ 63%`. This is why tail latency dominates at scale — see "the tail at scale" (Google, resources below).

---

## 5. Reducing Latency — Techniques

|Technique|How it helps|
|---|---|
|**Caching** (CDN, Redis, browser)|Avoids repeated expensive computation/network hops|
|**CDN / edge servers**|Serve content physically closer to the user|
|**Connection pooling / keep-alive**|Avoid repeated TCP/TLS handshake cost|
|**Async / non-blocking I/O**|Don't block threads waiting on slow I/O|
|**Database indexing**|Avoid full table scans|
|**Read replicas**|Serve reads from a nearby/less-loaded copy|
|**Batching**|Reduce per-request overhead by grouping|
|**Compression**|Smaller payloads = faster transfer (trade-off: CPU cost)|
|**HTTP/2 / HTTP/3 (QUIC)**|Multiplexing, reduced handshake overhead|
|**Precomputation / materialized views**|Move work out of the request path|

---

## 6. Latency vs Throughput — Trade-off

Increasing **batch size** often _improves throughput_ (more work done per unit time) but _increases latency per individual item_ (has to wait for the batch to fill). This is the classic trade-off in message queues, batch APIs, and network packet coalescing (e.g. Nagle's algorithm). See [[throughput]] for the full comparison.

---

## 7. Interview Talking Points

- Always ask/clarify: "latency requirement — what's the target p99, not just average?"
- Distinguish tail latency amplification in fan-out/microservices architectures — this is a favorite follow-up question.
- Tie latency-reduction techniques to the actual bottleneck identified (don't just say "add caching" without saying _what_ is slow and _why_ caching fixes it).

---

## 8. Resources

- [The Tail at Scale – Dean & Barroso, Google (PDF)](https://research.google/pubs/pub40801/)
- [Latency Numbers Every Programmer Should Know (interactive)](https://colin-scott.github.io/personal_website/research/interactive_latency.html)
- [Cloudflare Learning – What is Latency?](https://www.cloudflare.com/learning/performance/glossary/what-is-latency/)
- [High Scalability – Latency vs Throughput](http://highscalability.com/blog/2015/10/12/making-sense-of-performance-in-linux-systems.html)

---

## Related Notes

- [[throughput]]
- [[scalability]]
- [[system-design-networking-index]]