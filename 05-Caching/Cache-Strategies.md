# Cache Strategies (Read/Write Patterns)

## 1. Overview

The strategy determines **when data enters/leaves the cache relative to reads and writes**. Choosing the right one is a core system design decision — each has different consistency, latency, and complexity trade-offs. Prerequisite: [[caching]].

---

## 2. Cache-Aside (Lazy Loading) — Most Common

```
Read:  App checks cache → miss → App reads DB → App writes result to cache → returns
Write: App writes directly to DB → (cache untouched, or explicitly invalidated)
```

```python
def get_user(id):
    data = cache.get(f"user:{id}")
    if data is None:
        data = db.query("SELECT * FROM users WHERE id = ?", id)
        cache.set(f"user:{id}", data, ttl=300)
    return data
```

|Pros|Cons|
|---|---|
|Only requested data gets cached (no wasted memory)|First request after a miss is slow (cache miss penalty)|
|Cache failure doesn't break the system — falls back to DB|Risk of stale data if writes don't invalidate properly (see [[cache-invalidation]])|
|Simple to reason about|Extra round trip on every miss (check cache, then DB)|

**Most widely used pattern** — Redis + app-level caching in front of a DB is almost always cache-aside.

---

## 3. Read-Through

```
Read: App asks the CACHE for data → cache itself fetches from DB on miss → returns
```

Similar to cache-aside, but the **caching layer/library handles the DB fetch**, not the application code. App only ever talks to the cache.

|Pros|Cons|
|---|---|
|Simpler application code (cache abstracts the DB)|Requires a caching library/layer that supports this (not all do natively)|
|Consistent loading logic in one place|Same first-miss latency penalty as cache-aside|

---

## 4. Write-Through

```
Write: App writes to CACHE → cache synchronously writes to DB → confirms to app
```

Cache and DB are updated **together**, on every write.

|Pros|Cons|
|---|---|
|Cache is always consistent with DB (no stale reads)|Every write pays cache + DB latency (slower writes)|
|Good for read-heavy workloads where data must always be fresh|Data written but never read still gets cached (wasted memory for cold data)|

---

## 5. Write-Behind (Write-Back)

```
Write: App writes to CACHE → cache acknowledges immediately → cache asynchronously flushes to DB later (batched)
```

|Pros|Cons|
|---|---|
|Very fast writes (don't wait for DB)|Risk of **data loss** if cache crashes before flushing to DB|
|Can batch/coalesce multiple writes to reduce DB load|More complex to implement correctly (flush ordering, retry logic)|
|Good for write-heavy workloads (e.g. view counters, analytics events)|Weaker durability guarantee — a real trade-off against [[acid]]'s Durability|

---

## 6. Write-Around

```
Write: App writes directly to DB, bypassing cache entirely
Read: Cache-aside pattern applies on next read (miss → populate)
```

|Pros|Cons|
|---|---|
|Avoids caching data that's written but rarely read again|First read after write is always a cache miss (slower)|
|Prevents cache from filling with write-heavy, rarely-read data|Not ideal if the same data is read again soon after writing|

---

## 7. Side-by-Side Comparison

|Strategy|Write latency|Read latency (after write)|Consistency risk|Data loss risk|
|---|---|---|---|---|
|Cache-aside|Fast (DB only)|Slow on first read (miss)|Medium (needs invalidation)|None|
|Read-through|N/A (read pattern)|Slow on first read (miss)|Medium|None|
|Write-through|Slower (cache + DB)|Fast (always fresh)|Low|None|
|Write-behind|Fastest|Fast|Low (once synced)|**Higher** (async flush)|
|Write-around|Fast (DB only)|Slow on first read|Low for write-heavy data|None|

---

## 8. Choosing a Strategy (Interview Framework)

1. **Read-heavy, tolerate brief staleness?** → Cache-aside (most common default)
2. **Read-heavy, need cache always fresh?** → Write-through
3. **Write-heavy, can tolerate small data-loss risk for speed?** → Write-behind (e.g. incrementing a like/view counter)
4. **Data is written often but read rarely right after?** → Write-around
5. Often combined: e.g. **cache-aside for reads + write-through for critical fresh-read paths**

---

## 9. Interview Talking Points

- Name the pattern explicitly (don't just say "we'll cache it") — interviewers specifically listen for cache-aside vs write-through vs write-behind terminology.
- Always pair the strategy choice with the **consistency/durability trade-off** it implies — ties directly to [[consistency]] and [[acid]] (Durability).
- Mention TTL as a safety net regardless of strategy — even with a good invalidation strategy, a TTL bounds the worst-case staleness window (see [[cache-invalidation]]).

---

## 10. Resources

- [AWS – Caching Strategies (ElastiCache docs)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Strategies.html)
- [Redis Docs – Caching Patterns](https://redis.io/docs/latest/develop/get-started/data-store/)
- [ByteByteGo – Caching Patterns explained](https://bytebytego.com/)

---

## Related Notes

- [[caching]]
- [[cache-invalidation]]
- [[cache-stampede]]
- [[redis]]
- [[consistency]]