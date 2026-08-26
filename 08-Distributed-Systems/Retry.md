# Retry

## 1. Definition

Retry is the practice of **re-attempting a failed operation** instead of giving up immediately, on the assumption that many failures are **transient** (temporary network blip, momentary overload, brief unavailability) rather than permanent. A core [[fault-tolerance]] technique — but only safe to apply liberally when the operation is [[idempotency|idempotent]], and only useful when paired with a sensible [[timeout]] so you know when to stop waiting and retry in the first place.

---

## 2. When to Retry (and When Not To)

|Failure type|Retry?|
|---|---|
|Network timeout / connection reset|Yes — classic transient failure|
|`503 Service Unavailable` / `429 Too Many Requests`|Yes — server is explicitly signaling "try again later" (often with a `Retry-After` header — see [[rate-limiting]])|
|`500 Internal Server Error`|Sometimes — depends if it's transient (e.g. a momentary DB hiccup) or a real bug that will fail identically every time|
|`400 Bad Request` / `404 Not Found` / `401 Unauthorized`|**No** — retrying an invalid request just wastes resources; the request itself is wrong, not the environment|
|Circuit breaker is **open**|No — see [[circuit-breaker]]; retrying a known-down dependency defeats the purpose of the breaker|

**Rule of thumb:** retry on failures that are plausibly transient and where the client-side error category suggests "try again might work" (timeouts, 5xx, 429) — never retry on failures that indicate the request itself was wrong (4xx other than 429).

---

## 3. Backoff Strategies

Retrying immediately in a tight loop is dangerous — it can turn a brief blip into a sustained overload (a **retry storm**) by hammering an already-struggling dependency with a burst of repeated requests right when it can least handle it.

|Strategy|Behavior|Trade-off|
|---|---|---|
|**Fixed interval**|Retry every N seconds|Simple, but many clients retry in lockstep — still causes synchronized spikes|
|**Exponential backoff**|Wait doubles each attempt (1s, 2s, 4s, 8s...)|Gives the dependency increasing breathing room, but many clients still align on the same schedule|
|**Exponential backoff + jitter**|Exponential backoff with randomized variance added to each wait|**The production standard** — jitter spreads retries out over time instead of every client retrying in the same instant, preventing thundering-herd/retry-storm effects|

```python
import random

def backoff_with_jitter(attempt, base=1, cap=30):
    exp = min(cap, base * (2 ** attempt))
    return random.uniform(0, exp)  # "full jitter" — AWS's recommended approach
```

AWS's Architecture Blog popularized "full jitter" (random value between 0 and the exponential cap) as outperforming both fixed backoff and backoff without jitter under load — worth citing by name in an interview.

---

## 4. Bounding Retries

An unbounded retry loop just delays failure while wasting resources — always bound it:

- **Max attempt count** — stop after N tries (e.g. 3–5) and surface the failure
- **Max total elapsed time** — stop retrying once a deadline is exceeded, even if attempt count isn't reached (important for user-facing requests with a latency budget)
- **Combine with a [[circuit-breaker]]** — stop retrying entirely once the breaker trips open, rather than continuing to retry into a dependency already known to be down

---

## 5. Retry Placement in the Stack

Retries can happen at multiple layers — important to know where, so you don't accidentally stack them and cause a **retry amplification** problem (client retries 3x, and each of those calls a service that itself retries 3x downstream = 9x load):

- **Client-side** (SDKs, mobile apps) — closest to the user, but least visibility into system-wide load
- **API Gateway / load balancer** — centralizes retry policy, but less context about the specific failure
- **Service mesh (Envoy/Istio sidecar)** — declarative, consistent across services without code changes
- **Message queue redelivery** ([[rabbitmq]], [[kafka]] consumer reprocessing) — retry is implicit in at-least-once delivery semantics

Best practice: pick **one layer** to own retry logic for a given call path, and make the others aware of it (or explicitly disable retry at the other layers) to avoid amplification.

---

## 6. Retry + Idempotency: The Non-Negotiable Pairing

Retrying a non-idempotent operation (e.g. `POST /charge-card` with no dedup key) risks the exact failure mode retries are meant to avoid — the original request may have actually succeeded, and the retry duplicates the side effect (double charge, duplicate order). See [[idempotency]] for the idempotency-key pattern that makes retrying `POST`-style operations safe.

---

## 7. Interview Talking Points

- Immediately pair "we'd retry" with "idempotent operations only" and "exponential backoff with jitter" — these two qualifiers signal you're not just describing a naive loop
- Name the specific failure categories worth retrying (timeouts, 5xx, 429) vs not (4xx) — shows you're not retrying blindly
- Bring up retry amplification when asked about retries across a multi-hop service call chain — a strong differentiator most candidates miss
- Mention pairing retries with a [[circuit-breaker]] and a bounded [[timeout]] — retry alone, without these, can make an outage worse instead of better

---

## 8. Resources

- [AWS Architecture Blog – Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [Google SRE Book – Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
- [Microsoft – Retry Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry)

---

## Related Notes

- [[timeout]]
- [[idempotency]]
- [[circuit-breaker]]
- [[fault-tolerance]]
- [[rate-limiting]]
- [[kafka]]
- [[rabbitmq]]