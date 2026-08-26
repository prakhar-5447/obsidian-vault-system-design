# Timeout

## 1. Definition

A timeout is a **bound on how long a caller will wait for an operation to complete** before giving up and treating it as a failure. Without timeouts, a slow or hung dependency can hold a caller's resources (threads, connections) indefinitely, which is one of the most common root causes of cascading failure in distributed systems — the caller doesn't just wait, it eventually runs out of capacity to serve _any_ request. Foundational partner to [[retry]] and [[circuit-breaker]] in the resilience toolkit under [[fault-tolerance]].

---

## 2. Why "No Timeout" Is Dangerous

```
Caller ──► Dependency (hung, never responds)
Caller's thread/connection is held forever, waiting
→ Caller's own resource pool (threads, connections) fills up
→ Caller can no longer serve ANY request, even unrelated ones
→ Caller now looks "down" to ITS callers, and the failure propagates upward
```

This is the mechanism behind many real-world cascading outages: one slow dependency, no timeout, and the failure climbs the call stack. A timeout converts "wait forever" into "fail fast after a bound," which is a strict improvement even though it _feels_ like giving up — see [[circuit-breaker]] for the next step (stop calling entirely once timeouts keep happening).

---

## 3. Types of Timeouts

|Timeout type|What it bounds|
|---|---|
|**Connection timeout**|How long to wait while establishing a connection (TCP handshake, TLS negotiation) before giving up|
|**Read/response timeout**|How long to wait for a response after the request has been sent|
|**Request timeout (end-to-end)**|Total time budget for the entire operation, including connection + processing + response|
|**Idle timeout**|How long a connection can sit unused (no data flowing) before being closed — mainly for connection pool hygiene|

A well-behaved client usually needs **at least connection + read timeouts** set independently — a hung DNS lookup or handshake is a different failure mode than a slow response body, and conflating them into one generic "timeout" setting loses useful signal.

---

## 4. Choosing a Timeout Value

- **Too short** → healthy-but-slightly-slow requests get killed unnecessarily, causing needless [[retry]] attempts and wasted work (a self-inflicted reliability problem)
- **Too long** → defeats the purpose; resources stay tied up nearly as long as having no timeout at all
- **Base it on the dependency's actual latency distribution** — e.g. set the timeout around the p99 or p99.9 latency of the dependency under normal conditions, not a round guessed number
- **Budget timeouts across a call chain** — if service A calls B calls C, A's total timeout must exceed B's timeout plus B's own downstream budget for C, or A will time out _before_ B even finishes failing — a subtle bug worth naming explicitly in an interview
- Timeouts should generally be **stricter (shorter) the closer you are to the user** — an end-user-facing request has a tight latency budget; an internal batch job can tolerate a much longer timeout

---

## 5. Timeout Propagation (Deadlines)

Instead of each service in a call chain independently guessing its own timeout, a better pattern is passing a **deadline** (an absolute point in time, e.g. "this request must finish by 14:32:05.000") down through the call chain:

- Each hop computes its remaining budget as `deadline - now()` rather than using a fixed local timeout
- If the deadline has already passed by the time a downstream call would start, that hop can skip the call entirely and fail fast, saving wasted work
- gRPC has native deadline propagation support; many service meshes support this too — worth naming if the question involves a multi-hop internal architecture

---

## 6. Timeout's Relationship to Other Patterns

|Pattern|Relationship|
|---|---|
|[[retry]]|A timeout is what triggers a retry decision — no timeout means you never know _when_ to give up and retry|
|[[circuit-breaker]]|Repeated timeouts are one of the standard triggers for tripping a circuit open (some implementations treat "too slow" the same as "failed")|
|[[fault-tolerance]] – Bulkheads|Timeouts limit _how long_ a resource is held; bulkheads limit _how much_ of a resource pool one dependency can consume — complementary, not redundant|
|[[rate-limiting]]|Different axis — rate limiting bounds _how many_ requests are allowed; timeout bounds _how long_ an individual request is allowed to take|

---

## 7. Interview Talking Points

- Lead with the cascading-failure mechanism (resource exhaustion from hung calls) — this is the _why_, and interviewers want to hear it stated explicitly, not just "timeouts are good practice"
- Distinguish connection vs read/response timeout when asked to configure an HTTP client — a common follow-up probe
- Bring up the call-chain budget problem (A's timeout must exceed B + C's combined budget) unprompted — this is a detail that signals real production experience
- Mention deadline propagation as the more sophisticated alternative to independently-configured per-hop timeouts, especially if the system in question is a deep microservices chain

---

## 8. Resources

- [Google SRE Book – Handling Overload (timeouts section)](https://sre.google/sre-book/handling-overload/)
- [gRPC – Deadlines](https://grpc.io/docs/guides/deadlines/)
- [AWS Builders' Library – Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)

---

## Related Notes

- [[retry]]
- [[circuit-breaker]]
- [[fault-tolerance]]
- [[rate-limiting]]
- [[idempotency]]