# Fault Tolerance

## 1. Definition

Fault tolerance is a system's ability to **continue operating correctly (possibly in a degraded mode) when part of it fails**, rather than failing entirely. Distinct from **fault avoidance** (trying to prevent failures) — fault tolerance assumes failures _will_ happen (hardware dies, networks partition, processes crash) and designs for graceful survival rather than perfection. Umbrella concept that [[circuit-breaker]], [[distributed-locks]], replication, and redundancy all serve.

---

## 2. Types of Faults

|Type|Example|
|---|---|
|**Crash fault**|A node stops entirely and doesn't respond (the easiest to reason about)|
|**Omission fault**|A node fails to send/receive some messages but is otherwise alive|
|**Timing fault**|A node responds, but too slowly (violates a timing assumption)|
|**Byzantine fault**|A node behaves arbitrarily/maliciously — sends conflicting or incorrect information to different peers (hardest to tolerate; relevant in blockchain/adversarial contexts, less common in typical internal system design interviews)|

Most system-design-interview fault tolerance discussion assumes **crash faults** — simpler and covers the large majority of real production failures (node dies, disk dies, AZ goes down, network partitions).

---

## 3. Core Techniques

### Redundancy / Replication

Run multiple copies of a component so one failing doesn't take down the service. Applies at every layer: multiple app server instances behind a [[load-balancing|load balancer]], database replicas, multi-AZ/multi-region deployment.

### Failover

Automatic (or manual) switch to a healthy redundant instance when the active one fails.

- **Active-passive** — standby instance takes over only on failure (simpler, has a brief failover gap)
- **Active-active** — all instances handle traffic simultaneously (no failover gap, but requires handling concurrent writes/[[consistency]])

### Health Checks & Heartbeats

Continuous monitoring so failures are detected quickly — the faster a failure is detected, the faster failover/recovery can happen. Load balancers and orchestrators (Kubernetes) use these to route traffic only to healthy instances and restart unhealthy ones.

### Timeouts, Retries, Circuit Breakers

Bound how long you wait on a dependency, retry transient failures (with backoff and jitter to avoid a retry storm), and stop calling a dependency that's clearly down — see [[circuit-breaker]] for the full pattern.

### Bulkheads

Isolate resources (thread pools, connection pools) per dependency so one failing/slow dependency can't exhaust resources needed by calls to healthy dependencies — named after ship compartments that contain flooding to one section.

### Graceful Degradation

When a non-critical dependency fails, serve a reduced experience instead of failing the whole request (e.g. show a product page without personalized recommendations if the recommendation service is down, rather than a 500 error).

### Data Replication + Consensus

Systems like Raft/Paxos tolerate up to `(N-1)/2` node failures (of N total) while still making progress and staying consistent — this is the formal foundation under most "3 replicas can survive 1 failure" style guarantees. See [[consistency]] and [[distributed-locks]] for where consensus shows up.

---

## 4. Quantifying Fault Tolerance

|Metric|Meaning|
|---|---|
|**Availability**|% of time the system is operational — often expressed in "nines" (99.9% = ~8.7 hrs downtime/year, 99.99% = ~52 min/year)|
|**MTBF (Mean Time Between Failures)**|Average time between failures — higher is better|
|**MTTR (Mean Time To Recovery)**|Average time to recover from a failure — lower is better|
|**RTO (Recovery Time Objective)**|Target maximum time to restore service after a disaster|
|**RPO (Recovery Point Objective)**|Target maximum acceptable data loss, measured in time (e.g. "RPO of 5 minutes" = at most 5 minutes of data since the last backup/replication point can be lost)|

**Availability is a function of both MTBF and MTTR** — you can improve availability either by failing less often or by recovering faster; in practice, investing in fast detection/recovery (lower MTTR) is often cheaper and more effective than chasing ever-higher MTBF.

---

## 5. Redundancy Trade-offs

|Approach|Pro|Con|
|---|---|---|
|N+1 redundancy (one spare)|Cheap, covers single failure|Doesn't cover simultaneous failures|
|Multi-AZ|Survives a datacenter-level outage|Doesn't survive a full regional outage|
|Multi-region|Survives regional disasters|Expensive, cross-region [[consistency]] complexity, higher write latency if strongly consistent|

The right level of redundancy is a **cost vs risk-tolerance decision**, not a technical maximum — always worth stating this explicitly in an interview rather than defaulting to "just replicate everything everywhere."

---

## 6. Interview Talking Points

- Name faults you're assuming (usually crash faults) explicitly — shows precision rather than a vague "handle failures" answer
- Structure the answer by layer: redundancy (don't have a single point of failure), detection (health checks), reaction (failover, circuit breaking, retries), and degradation (fallback behavior) — this is a strong default framework for almost any "how would you make this resilient" question
- Bring up RTO/RPO when the question touches disaster recovery specifically, not just single-node failure
- Connect MTTR explicitly — good observability (fast detection) is itself a fault-tolerance lever, not just an operational nicety

---

## 7. Resources

- [Google SRE Book – Chapter on Reliability](https://sre.google/sre-book/table-of-contents/)
- [AWS – Reliability Pillar (Well-Architected Framework)](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)
- [Microsoft – Resiliency Patterns](https://learn.microsoft.com/en-us/azure/architecture/patterns/category/resiliency)

---

## Related Notes

- [[circuit-breaker]]
- [[consistency]]
- [[eventual-consistency]]
- [[distributed-locks]]
- [[reliability]]
- [[rate-limiting]]