# Caching — Overview

## 1. Definition

Caching = storing a **copy of expensive-to-compute or expensive-to-fetch data** in a faster-access layer, so future requests for the same data are served quickly instead of redoing the work.

It's the single highest-leverage technique for improving [[latency]] and [[throughput]], and for reducing load on downstream systems (databases, APIs).

---

## 2. Where Caching Happens (Every Layer of the Stack)

```
Client (browser cache)
   ↓
CDN / Edge cache
   ↓
Load Balancer / Reverse Proxy cache (e.g. Varnish, Nginx)
   ↓
Application-level cache (Redis, Memcached)
   ↓
Database query cache / buffer pool
   ↓
Disk
```

| Layer                             | Example                                       | What it caches                                            |
| --------------------------------- | --------------------------------------------- | --------------------------------------------------------- |
| **Client-side**                   | Browser cache, mobile app local storage       | Static assets, API responses                              |
| **CDN/Edge**                      | Cloudflare, CloudFront                        | Static files, sometimes API responses (edge caching)      |
| **Reverse proxy**                 | Nginx, Varnish                                | Full HTTP responses                                       |
| **Application/distributed cache** | [[redis]], Memcached                          | DB query results, session data, computed objects          |
| **Database-level**                | PostgreSQL buffer pool, MySQL query cache     | Recently accessed pages/rows in memory                    |
| **In-process/local**              | In-memory dict/[[lru]] cache within a service | Per-instance hot data, avoids even a network hop to Redis |

**Interview point:** the closer the cache is to the user, the lower the latency, but the harder it is to keep consistent/invalidate — this tension runs through every layer above.

---

## 3. Cache Hit vs Cache Miss

- **Hit**: requested data found in cache → fast response
- **Miss**: not found → fetch from source of truth (DB/API), often populate cache for next time (see [[cache-strategies]])

**Hit ratio** = hits / (hits + misses) — the primary metric for cache effectiveness. A low hit ratio means the cache isn't earning its complexity/cost.

---

## 4. What Makes Data Cacheable

Good candidates:

- **Read-heavy, write-light** data (product catalogs, user profiles, config)
- **Expensive to compute/fetch** (complex joins, aggregations, external API calls)
- **Tolerant of slight staleness** (view counts, recommendation lists)

Poor candidates:

- Highly volatile data with strict consistency needs (real-time stock prices for a trading engine, account balances mid-transaction)
- Data accessed only once (no reuse = no benefit, just overhead)

---

## 5. Eviction Policies (What Gets Removed When Cache Is Full)

|Policy|How it works|Best for|
|---|---|---|
|**LRU** (Least Recently Used)|Evict the item not accessed for the longest time|General-purpose default, most common|
|**LFU** (Least Frequently Used)|Evict the item accessed least often|Workloads with a stable "hot set" that shouldn't be evicted by a temporary burst|
|**FIFO**|Evict oldest-inserted item, regardless of access|Simple, rarely optimal|
|**TTL-based**|Evict once a time-to-live expires, regardless of access pattern|Data with a natural expiry (session tokens, temporary results)|
|**Random**|Evict a random item|Very cheap, sometimes surprisingly competitive at scale (used internally by some systems)|

Redis supports several of these configurably — see [[redis]].

---

## 6. Sub-Topics (Deep Dives)

- **How** to keep cache and source-of-truth in sync → [[cache-invalidation]]
- **What pattern** to use for reading/writing through the cache → [[cache-strategies]]
- **What happens when the cache empties and everyone hits the DB at once** → [[cache-stampede]]
- **The most common tool used to implement all of this** → [[redis]]

---

## 7. Interview Talking Points

- Always justify _what_ you're caching and _why_ it's safe to be stale — vague "add a cache" answers are a red flag to interviewers.
- Mention **hit ratio** as the metric you'd monitor to validate a caching decision.
- Bring up the **latency layering diagram** above when asked "where would you cache this" — shows you think about the whole request path, not just "put Redis in front of the DB."

---

## 8. Resources

- _Designing Data-Intensive Applications_ by Martin Kleppmann — Ch. 3 (caching context within storage engines)
- [AWS – Caching Overview](https://aws.amazon.com/caching/)
- [Redis Docs – Introduction](https://redis.io/docs/latest/develop/get-started/)
- [High Scalability – Caching strategies](http://highscalability.com/)

---

## Related Notes

- [[cache-invalidation]]
- [[cache-strategies]]
- [[cache-stampede]]
- [[redis]]
- [[latency]]
- [[throughput]]
- [[cdn]]