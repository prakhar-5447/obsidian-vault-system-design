# Rate Limiting

## 1. Definition

Rate limiting restricts **how many requests a client can make** in a given time window, protecting backend systems from overload, abuse, and unfair resource consumption by any single client. Often implemented at the [[api-gateway]] or [[load-balancing|load balancer]] layer.

---

## 2. Why Rate Limit

- **Prevent abuse/DDoS** — stop a single bad actor from overwhelming the system
- **Fair usage** — prevent one heavy client from starving others of capacity
- **Cost control** — protect against runaway usage (especially for metered/billed downstream APIs)
- **Protect downstream dependencies** — a database or third-party API that can only handle X requests/sec needs the gateway to enforce that ceiling upstream
- **Tiered plans** — free vs paid API tiers with different allowed request rates

---

## 3. Rate Limiting Algorithms

### Fixed Window Counter

```
Count requests in a fixed time window (e.g. per-minute buckets). Reset counter each new window.
```

|Pros|Cons|
|---|---|
|Simple, cheap (one counter + TTL)|**Boundary burst problem**: a client can send max requests at the end of one window AND max requests at the start of the next — effectively 2x the limit in a short burst around the boundary|

### Sliding Window Log

```
Store a timestamp for every request; count timestamps within the last N seconds on each new request.
```

|Pros|Cons|
|---|---|
|Precise — no boundary burst issue|Memory-heavy (stores every request timestamp); expensive at high volume|

### Sliding Window Counter (Hybrid — Most Common in Practice)

```
Weighted average of current + previous fixed window counts, based on how far into the current window we are.
```

Approximates the sliding log's accuracy with the fixed window's low memory cost — this is what most production systems (including Cloudflare) actually use.

### Token Bucket — Most Widely Used

```
Bucket holds up to N tokens, refilled at a fixed rate (e.g. 10 tokens/sec).
Each request consumes 1 token. No tokens left → request rejected (or queued).
```

|Pros|Cons|
|---|---|
|Allows **controlled bursts** (if bucket is full, a burst up to N requests is allowed instantly) while enforcing a steady average rate|Slightly more complex than fixed window|
|Widely implemented (AWS API Gateway, Stripe, many libraries)||

**Why bursts matter:** real traffic isn't perfectly smooth — a token bucket lets a client briefly exceed the average rate (using saved-up tokens) without being unfairly throttled, while still capping the long-run rate.

### Leaky Bucket

```
Requests enter a queue (the "bucket") and are processed at a fixed, constant output rate, regardless of input burstiness.
```

|Pros|Cons|
|---|---|
|Smooths bursts into a constant outflow rate — protects downstream from spikes entirely|Bursts get queued/delayed rather than allowed — worse UX if the client's burst was legitimate|

**Token bucket vs leaky bucket (classic interview question):** token bucket allows bursts up to bucket capacity; leaky bucket enforces a strictly constant output rate regardless of input pattern. Token bucket is generally preferred for API rate limiting (allows legitimate bursts); leaky bucket is more common for traffic shaping/smoothing (e.g. network QoS).

---

## 4. Comparison Table

|Algorithm|Burst handling|Memory cost|Accuracy|Common use|
|---|---|---|---|---|
|Fixed window|Poor (boundary issue)|Very low|Low|Simple internal limits|
|Sliding window log|N/A (very precise)|High|Highest|Low-volume, precision-critical|
|Sliding window counter|Good approximation|Low|High|Production APIs (Cloudflare, etc.)|
|Token bucket|Allows controlled bursts|Low|High|Most public API rate limiting|
|Leaky bucket|Smooths, doesn't allow|Low|High|Traffic shaping, queue-based smoothing|

---

## 5. What to Rate Limit By (Keying Strategy)

|Key|Use case|
|---|---|
|**Per-user / API key**|Standard API tier enforcement|
|**Per-IP**|Protect against anonymous abuse (login attempts, public endpoints)|
|**Per-endpoint**|Some endpoints are more expensive (e.g. search, report generation) and need tighter limits than others|
|**Global**|Protect a shared downstream dependency (e.g. total requests to a third-party API across all users)|

Real systems often combine multiple layers — e.g. per-IP limit AND per-user limit AND a global ceiling simultaneously.

---

## 6. Implementation: Redis as the Rate Limiter Backend

Rate limiting needs a shared, fast counter accessible by all gateway/service instances — [[redis]] is the standard choice:

```python
# Simple fixed-window with Redis
key = f"ratelimit:{user_id}:{current_minute}"
count = redis.incr(key)
if count == 1:
    redis.expire(key, 60)
if count > LIMIT:
    return 429  # Too Many Requests
```

**Token bucket in Redis** — typically implemented via a Lua script (for atomicity of the check-and-decrement) or Redis's built-in `CL.THROTTLE` (RedisCell module).

---

## 7. Distributed Rate Limiting Challenges

- **Single shared counter (Redis)** — simplest, but Redis becomes a dependency every request must hit (adds latency, and is itself a potential bottleneck/SPOF unless made highly available — see [[redis]] scaling section)
- **Local/in-memory per-node limiting** — no shared state needed, but inaccurate at the edge (N nodes × local limit ≠ true global limit) — good enough when slight over-allowance under high node count is acceptable
- **Approximate global limiting** — some systems accept slight inaccuracy in exchange for avoiding a synchronous dependency on every request (e.g. periodically syncing local counts)

---

## 8. Response to a Rate-Limited Client

- Return **HTTP 429 Too Many Requests** (see [[http]] status codes)
- Include headers so clients can self-regulate:
    
    ```
    X-RateLimit-Limit: 100X-RateLimit-Remaining: 42X-RateLimit-Reset: 1699999999Retry-After: 30
    ```
    
- Well-behaved clients back off automatically based on `Retry-After` — same exponential backoff pattern discussed in [[message-queues]]

---

## 9. Interview Talking Points

- Name the algorithm explicitly (token bucket is the safe, most broadly correct default answer) and explain _why_ it fits — bursts are usually legitimate and shouldn't be punished like sustained abuse.
- Mention **where** rate limiting is enforced (API Gateway is the natural place — see [[api-gateway]]) and **what** it's keyed by (per-user, per-IP, per-endpoint, often combined).
- Bring up **Redis + Lua script** as the concrete distributed implementation — shows you can go from concept to real architecture.
- Address the **distributed accuracy trade-off** if asked about scaling the rate limiter itself across many gateway instances.

---

## 10. Resources

- [Cloudflare – How we built rate limiting (sliding window)](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/)
- [Stripe Engineering – Scaling your API with rate limiters](https://stripe.com/blog/rate-limiters)
- [Redis Docs – Rate limiting patterns](https://redis.io/glossary/rate-limiting/)
- [System Design Primer – Rate limiting](https://github.com/donnemartin/system-design-primer)

---

## Related Notes

- [[api-gateway]]
- [[redis]]
- [[reliability]]
- [[http]]
- [[message-queues]]