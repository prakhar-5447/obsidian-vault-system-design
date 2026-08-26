# Reliability

## 1. Definition

Reliability = the probability that a system performs its **intended function correctly**, for a specified period, under stated conditions.

It answers: _"When the system responds, is the response correct, and does the system keep working correctly over time — not just right now?"_

Not the same as [[availability]] — a system can be **up** (available) but returning **wrong data** (unreliable), e.g. corrupted results, dropped messages, silent data loss.

---

## 2. Reliability vs Availability vs Fault Tolerance

|Term|Question it answers|
|---|---|
|**Reliability**|Does it work correctly, consistently, over time?|
|**Availability**|Is it up and responding right now?|
|**Fault tolerance**|Does it keep working correctly _despite_ failures?|
|**Resilience**|Does it _recover_ well after failure?|

A car that starts every time (available) but sometimes has faulty brakes (unreliable) illustrates the distinction well.

---

## 3. Key Metrics

|Metric|Meaning|Formula|
|---|---|---|
|**MTBF**|Mean Time Between Failures — avg time system runs before failing|Total uptime / number of failures|
|**MTTR**|Mean Time To Recovery — avg time to restore service after failure|Total downtime / number of incidents|
|**MTTF**|Mean Time To Failure — for non-repairable components|Total operational time / units tested|

```
Availability = MTBF / (MTBF + MTTR)
```

This ties reliability metrics directly back to availability — **improving MTTR is often cheaper and faster than improving MTBF**, which is why incident response tooling (alerting, runbooks, auto-rollback) is a high-leverage investment.

---

## 4. How to Build Reliable Systems

| Technique                       | Purpose                                                               |
| ------------------------------- | --------------------------------------------------------------------- |
| **Redundancy / replication**    | No single point of failure                                            |
| **Retries with backoff**        | Handle transient failures gracefully                                  |
| **Idempotency**                 | Safe retries don't cause duplicate side effects (e.g. double charges) |
| **Checksums / data validation** | Catch corruption early                                                |
| **Write-ahead logs (WAL)**      | Durability — recover state after crash                                |
| **Graceful degradation**        | Partial functionality > total failure                                 |
| **Circuit breakers**            | Prevent cascading failures across services                            |
| **Chaos engineering**           | Proactively test failure modes (Netflix Chaos Monkey)                 |
| **Automated rollback**          | Fast recovery from bad deploys                                        |

---

## 5. Failure Types to Reason About

- **Crash failure**: component stops entirely
- **Omission failure**: messages/requests silently dropped
- **Timing failure**: response too slow (violates latency SLA even if "correct")
- **Byzantine failure**: component behaves arbitrarily/maliciously (wrong data, inconsistent responses to different observers) — relevant in blockchain/distributed consensus contexts

---

## 6. Reliability Patterns in Distributed Systems

- **Replication** — same data on multiple nodes; survives node failure
- **Checkpointing** — periodic state snapshots to recover from
- **Quorum reads/writes** — require majority agreement to tolerate node failure without losing correctness
- **Dead-letter queues** — failed messages aren't lost, they're captured for inspection/retry
- **Idempotent APIs** — client retries (due to timeout) don't cause duplicate processing (e.g. using idempotency keys in payment APIs)

---

## 7. Interview Talking Points

- Always tie reliability improvements to a **cost/complexity trade-off** — five-nines reliability is expensive.
- When asked "how do you ensure reliable message delivery," mention: at-least-once vs exactly-once vs at-most-once delivery semantics, and how idempotency solves the at-least-once duplicate problem cheaply.
- Reliability and [[consistency]] intersect: a "reliable" system that returns stale/incorrect data during failover isn't fully reliable from the user's perspective — call this out.

---

## 8. Resources

- [Google SRE Book – Ch. 1: Introduction](https://sre.google/sre-book/introduction/)
- [AWS Well-Architected – Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)
- [Netflix Tech Blog – Chaos Engineering](https://netflixtechblog.com/tagged/chaos-engineering)
- _Designing Data-Intensive Applications_ by Martin Kleppmann — Ch. 1 (Reliable, Scalable, Maintainable Applications)

---

## Related Notes

- [[availability]]
- [[cap-theorem]]
- [[consistency]]
- [[system-design-networking-index]]