# Cache Invalidation

## 1. Definition

Cache invalidation = the process of **removing or updating stale cached data** when the underlying source of truth changes. Famously one of the "two hard things in computer science" (Phil Karlton) — because it requires reasoning about **every place** data can change, and getting the timing right across distributed components.

Prerequisite: [[caching]], [[cache-strategies]].

---

## 2. Why It's Hard

- Data can change from **multiple write paths** (API, background job, admin panel, another service) — every path must trigger invalidation, or the cache silently goes stale
- **Distributed caches**: invalidating on one node doesn't automatically invalidate copies on other nodes/regions
- **Timing races**: a write and a cache refresh happening concurrently can leave the cache with an even-older value than before (see the race condition example below)
- No single "right" answer — the correct approach depends on how stale is acceptable for that specific data

---

## 3. Invalidation Strategies

### TTL (Time-To-Live) — Simplest, Most Common

```
cache.set("user:123", data, ttl=300)  # expires in 5 minutes
```

|Pros|Cons|
|---|---|
|Simple, no explicit invalidation logic needed|Data can be stale for up to the full TTL window|
|Self-healing (bad/stuck cache entries expire eventually)|Choosing the "right" TTL is a guess — too short = low hit ratio, too long = stale data|

**Almost always used as a safety net**, even alongside other strategies below.

### Explicit / Write-Triggered Invalidation

The write path actively deletes or updates the cache entry when data changes:

```python
def update_user(id, new_data):
    db.update("users", id, new_data)
    cache.delete(f"user:{id}")   # or cache.set() with new value
```

|Pros|Cons|
|---|---|
|Cache is fresh immediately after any write|Every write path must remember to invalidate — easy to miss one (e.g. a background job that updates the DB directly)|
|No unnecessary staleness window|More code paths to maintain and test|

**Delete vs update the cache entry:**

- **Delete (invalidate)** — next read repopulates it (cache-aside pattern) — safer, avoids race conditions
- **Update in place** — avoids the next read being a cache miss, but risks writing a stale value if writes race (see below)

### Event-Driven Invalidation (Pub/Sub / CDC)

For multi-service or multi-region caches, use an event stream so **all cache nodes** invalidate together:

```
DB write → CDC (e.g. Debezium) → Kafka topic "user-updated" → all cache nodes subscribe → each invalidates its local copy
```

|Pros|Cons|
|---|---|
|Works across distributed caches / multiple services|More infrastructure (message broker, CDC pipeline)|
|Decouples "who writes" from "who must invalidate"|Eventual, not instant — brief window of inconsistency across nodes|

### Versioned Keys / Cache Busting

Instead of invalidating, **change the key** so old cached entries are simply never looked up again:

```
cache key: "user:123:v7"   (v7 = version number, bumped on every update)
```

Common for **static assets** (`app.a1b2c3.js` — hash in filename) — old cached files just become unreachable, no explicit deletion needed, and CDNs/browsers can cache "forever" since the URL itself changes on update.

---

## 4. The Classic Race Condition (Read-Modify-Write Under Concurrency)

```
T1: reads DB (old value) ─────────────────┐
T2: writes DB (new value), invalidates cache│
T1: writes stale value into cache ─────────┘  ← now cache has OLD data, permanently (until TTL)
```

This happens when a cache-aside read populates the cache **after** a concurrent write already invalidated it. **Mitigations:**

- Short TTL as a safety net (bounds how long the stale value survives)
- **Delete-then-set with a version check** — only write to cache if the version matches what was read (optimistic concurrency, similar to [[transactions]]' OCC)
- Delay cache population briefly after invalidation (rarely used, adds complexity)

---

## 5. Invalidating Related/Derived Data

A single change can affect **many cached entries** (e.g. updating a user's name should invalidate their profile cache, any cached lists they appear in, any cached search results containing them).

**Approaches:**

- **Tag-based invalidation** — tag cache entries with related IDs, invalidate by tag (e.g. Redis doesn't do this natively, but application layers/CDNs like Cloudflare "cache tags" support it)
- **Coarser cache granularity** — cache at a level where fewer invalidation triggers are needed (trade-off: less fine-grained caching benefit)
- **Accept staleness for derived/aggregate data** — often the pragmatic answer (e.g. "trending posts" list refreshes every 60s regardless, no need for real-time invalidation)

---

## 6. Interview Talking Points

- Always propose **TTL + explicit invalidation together**, not one or the other — TTL is the safety net, explicit invalidation is the freshness optimization.
- When asked about multi-region/multi-service caching, bring up **event-driven invalidation via pub/sub or CDC** — a strong signal of distributed systems maturity.
- Mention the **read-modify-write race condition** explicitly if discussing cache correctness — it's a favorite "gotcha" follow-up question.
- For static assets, mention **versioned/hashed filenames** as the standard, elegant solution that sidesteps invalidation entirely.

---

## 7. Resources

- [Phil Karlton's famous quote — background/context (Martin Fowler mentions)](https://martinfowler.com/bliki/TwoHardThings.html)
- [AWS – Cache Invalidation Strategies](https://aws.amazon.com/caching/invalidation/)
- [Redis Docs – Expiry (TTL)](https://redis.io/docs/latest/commands/expire/)
- [Cloudflare – Cache Purge / Tags](https://developers.cloudflare.com/cache/how-to/purge-cache/)

---

## Related Notes

- [[caching]]
- [[cache-strategies]]
- [[cache-stampede]]
- [[redis]]
- [[consistency]]