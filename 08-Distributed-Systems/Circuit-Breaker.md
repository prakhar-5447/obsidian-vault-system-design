# Circuit Breaker

## 1. Definition

A circuit breaker is a resilience pattern that **stops a service from repeatedly calling a dependency that's already failing**, failing fast instead of piling up slow/failing requests. Named after electrical circuit breakers — trip open to stop current flow when something's wrong, rather than letting the fault propagate. Sits alongside retries, timeouts, and bulkheads as a core [[fault-tolerance]] technique.

---

## 2. Why It Matters

- **Prevents cascading failure** — without it, a slow/failing downstream service causes callers to pile up threads/connections waiting on it, which can exhaust the caller's own resources and take down services that were otherwise healthy (the classic microservices death spiral)
- **Fails fast** — rejecting a request immediately (with a clear error) is better for the caller than waiting out a timeout on a call that was going to fail anyway
- **Gives the failing dependency room to recover** — stops hammering an already-struggling service with more traffic, which can make its recovery harder

---

## 3. The Three States

```
        failure threshold exceeded
  CLOSED ─────────────────────────► OPEN
    ▲                                 │
    │                                 │ timeout elapses
    │ success                         ▼
    └──────────────────────────  HALF-OPEN
                 failure
                    │
                    ▼
                  OPEN
```

|State|Behavior|
|---|---|
|**Closed**|Normal operation — requests pass through to the dependency; failures are counted|
|**Open**|Requests are rejected immediately (no call made to the dependency) — fail fast; after a configured timeout, moves to Half-Open|
|**Half-Open**|A limited number of trial requests are allowed through to test if the dependency has recovered — success → Closed, failure → back to Open|

---

## 4. Tripping Logic (Closed → Open)

- **Failure count threshold** — trip after N consecutive failures
- **Failure rate threshold** — trip if failure rate exceeds X% over a rolling window (more common in production — avoids tripping on a couple of unlucky failures in low traffic)
- **Slow call threshold** — some implementations also trip on calls that exceed a latency threshold, treating "too slow" the same as "failed", since a hung call ties up resources just like an error does
- Libraries like **Resilience4j**, **Hystrix** (deprecated, but historically influential), and **Polly** (.NET) implement this with a sliding window of recent call outcomes

---

## 5. Circuit Breaker vs Related Patterns

|Pattern|Purpose|How it differs|
|---|---|---|
|**Retry**|Re-attempt a failed call|Circuit breaker stops retries from happening at all once a dependency is known-bad — retrying into an open circuit just wastes time; the two are usually combined (retry with backoff _inside_ a closed circuit, no retries once open)|
|**Timeout**|Bound how long a single call can take|Timeout protects one call; circuit breaker protects against _repeated_ calls to a failing dependency|
|**Bulkhead**|Isolate resources (e.g. thread pools) per dependency|Prevents one slow dependency from starving resources needed by calls to other dependencies — complements circuit breaking|
|**Rate limiting**|Limit how much traffic _you_ send/accept|Circuit breaker reacts to a dependency's _health_; [[rate-limiting]] enforces a _ceiling_ regardless of health|

These are typically layered together: rate limit inbound traffic, timeout + retry + circuit-break outbound calls, bulkhead resource pools per dependency.

---

## 6. Fallback Behavior (What Happens on Open)

An open circuit still needs to return _something_ to the caller — common strategies:

- **Fail fast with an error** — simplest, pushes the decision upstream
- **Return cached/stale data** — acceptable staleness beats no response (e.g. show last-known product price)
- **Return a default/degraded response** — e.g. show a generic recommendation list instead of a personalized one
- **Queue for later** — for non-time-sensitive writes, queue the request and process once the circuit closes again (only viable when eventual processing is acceptable — see [[eventual-consistency]])

---

## 7. Where It's Implemented

- **Client-side library** in the calling service (Resilience4j, Polly, Sentinel) — most common, gives fine-grained per-dependency control
- **Service mesh / sidecar** (Envoy, Istio) — circuit breaking configured declaratively at the infrastructure layer, no code changes needed per service
- **API Gateway** — can circuit-break calls to backend services centrally

---

## 8. Interview Talking Points

- Draw the three-state diagram — interviewers want to see you understand Half-Open exists specifically to test recovery without fully committing traffic back
- Explicitly connect it to **cascading failure prevention** — this is the "why," not just the mechanism
- Mention it's usually combined with retries, timeouts, and bulkheads, not used alone — shows you think in layered resilience, not a single silver-bullet pattern
- Be ready to discuss what a sensible fallback looks like for the specific system in the question (stale cache vs error vs degraded response)

---

## 9. Resources

- [Martin Fowler – CircuitBreaker](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Resilience4j Docs – CircuitBreaker](https://resilience4j.readme.io/docs/circuitbreaker)
- [Microsoft – Circuit Breaker Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
- [Netflix Tech Blog – Hystrix (historical)](https://netflixtechblog.com/introducing-hystrix-for-resilience-engineering-13531c1ab362)

---

## Related Notes

- [[fault-tolerance]]
- [[rate-limiting]]
- [[eventual-consistency]]
- [[reliability]]
- [[api-gateway]]