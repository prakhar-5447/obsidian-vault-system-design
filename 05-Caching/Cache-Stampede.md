# Cache Stampede (Thundering Herd)

## 1. Definition

Cache stampede (a.k.a. **thundering herd**, **dogpile effect**) = when a **popular cache entry expires or is invalidated**, and a large number of concurrent requests all miss the cache **at the same instant** and hammer the backend/database simultaneously to recompute the same value.

```
Cache entry for "homepage:trending" expires
   ↓
10,000 concurrent requests all miss the cache in the same millisecond
   ↓
10,000 identical DB queries fire simultaneously
   ↓
DB overloaded → slow/fails → more requests pile up (retries) → cascading failure
```

This is a classic real-world outage cause, especially for **high-traffic, hot keys** (viral content, popular product pages, homepage data).

---

## 2. Why It's Dangerous

- The backend was sized assuming the **cache absorbs most traffic** — a stampede exposes it to full unmitigated load all at once
- Can trigger a **cascading failure**: DB slows down → requests time out → clients retry → even more load → DB falls further behind (death spiral)
- Often invisible until it happens — works fine at low traffic, breaks catastrophically once a hot key exists at scale

---

## 3. Mitigation Strategies

### Mutex / Locking (Single-Flight Pattern)

Only **one** request recomputes the value; others wait for it or serve stale data meanwhile.

```python
def get_data(key):
    data = cache.get(key)
    if data is not None:
        return data

    lock_acquired = cache.set(f"lock:{key}", 1, nx=True, ttl=10)  # NX = only if not exists
    if lock_acquired:
        data = expensive_db_query()
        cache.set(key, data, ttl=300)
        cache.delete(f"lock:{key}")
        return data
    else:
        # another request is already recomputing — wait briefly and retry, or serve stale
        sleep(0.05)
        return get_data(key)
```

|Pros|Cons|
|---|---|
|Only 1 DB query instead of thousands|Requests behind the lock wait (added latency for them)|
|Simple to implement with Redis `SET NX`|Lock itself can become a bottleneck/SPOF if not careful|

### Probabilistic Early Expiration (Recompute Before It Fully Expires)

Instead of a hard TTL cutoff, each read has a small, **increasing probability of proactively recomputing** the value slightly _before_ actual expiry — spreading recomputation out over time instead of all at once.

```
probability of early recompute ∝ (time since last computed) / (TTL)
```

Used by Facebook's "XFetch" algorithm — a well-known, citable technique in interviews.

|Pros|Cons|
|---|---|
|Smooths out load — no single moment of mass expiry|Slightly more complex logic|
|No explicit locking needed|Some requests still do "unnecessary" early recompute|

### Serve Stale While Revalidating

Keep serving the **old (stale) value** while one background process refreshes it — clients never see a miss at all.

```
Cache-Control: stale-while-revalidate=60
```

| Pros                                                                   | Cons                                                                   |
| ---------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Zero latency spike for users — always get _a_ response                 | Users may see slightly stale(outdated) data during revalidation window |
| Very common in HTTP caching (`stale-while-revalidate` header) and CDNs | Requires background refresh logic                                      |

### Staggered/Jittered TTLs

Don't set every related cache entry to expire at exactly the same time — add random jitter to TTLs so they expire spread out, not all at once.

```python
ttl = base_ttl + random.randint(0, 60)  # jitter to avoid synchronized expiry
```

Especially important when a deploy/cache-warm event sets many keys with the **same** TTL simultaneously — they'd all expire together later without jitter.

### Precomputation / Never Let It Expire Organically

For known hot keys (e.g. homepage, trending list), **proactively refresh on a schedule** (background job) rather than relying on lazy expiration-triggered recomputation at all — the cache is never actually "cold" for that key.

---

## 4. Comparison of Approaches

|Strategy|Complexity|Latency impact for users|Best for|
|---|---|---|---|
|Mutex/locking|Medium|Some requests wait|General-purpose, moderate traffic|
|Probabilistic early expiration|Medium|None (smoothed)|High-traffic hot keys|
|Stale-while-revalidate|Low-Medium|None (always serves something)|Content that's OK to be briefly stale|
|Jittered TTLs|Low|None (prevention, not reaction)|Any system with many keys set at once|
|Scheduled precomputation|Medium|None|Known hot/critical keys (homepage, config)|

**Interview default answer:** combine **jittered TTLs** (cheap prevention) + **mutex or stale-while-revalidate** (reactive safety net) — this covers both the common case and the edge case.

---

## 5. Related Concept: Cache Penetration

A related but distinct problem: requests for a key that **doesn't exist at all** (not even in the DB) — every such request bypasses the cache entirely and hits the DB, since there's nothing to cache. Can be exploited (e.g. requesting random non-existent IDs to bypass caching as a DoS vector).

**Mitigation:** cache the **negative result** too (`null`/`not found`) with a short TTL, or use a **Bloom filter** to quickly check "does this key possibly exist" before even querying the DB.

---

## 6. Interview Talking Points

- Bring this up specifically when discussing **hot keys/viral content** in a caching design — a strong differentiator most candidates miss.
- Name the mitigation explicitly (mutex/locking, probabilistic early expiry, stale-while-revalidate) rather than vaguely saying "handle concurrent misses."
- Mention **cache penetration** and negative caching/Bloom filters as the adjacent problem — shows breadth.

---

## 7. Resources

- [Facebook Research – XFetch: Reducing Cache Stampedes (PDF)](https://research.facebook.com/publications/xfetch-reducing-the-cost-of-cache-stampedes/)
- [Cloudflare – Stale-While-Revalidate explainer](https://developers.cloudflare.com/cache/concepts/stale-while-revalidate/)
- [Redis Docs – Distributed Locks (Redlock)](https://redis.io/docs/latest/develop/use/patterns/distributed-locks/)
- [MDN – Cache-Control: stale-while-revalidate](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control#stale-while-revalidate)

---

## Related Notes

- [[caching]]
- [[cache-invalidation]]
- [[cache-strategies]]
- [[redis]]
- [[reliability]]