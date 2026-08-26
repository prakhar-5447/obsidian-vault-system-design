# Eventual Consistency

## 1. Definition

Eventual consistency is a weak [[consistency]] model guaranteeing that **if no new writes occur, all replicas will eventually converge to the same value** — with no bound on how long convergence takes. It's the model most **AP** systems (in [[consistency|CAP terms]]) choose: prioritize availability and low latency over every read reflecting the latest write. Widely used for data where brief staleness is an acceptable trade for speed and uptime.

---
## 2. Why Choose It

- **Availability** — replicas can serve reads/writes even during a network partition, without waiting to coordinate with other nodes
- **Lower latency** — no cross-replica coordination needed on the write or read path
- **Higher throughput** — writes can be acknowledged locally and propagated asynchronously, rather than waiting on quorum agreement
- **Better multi-region performance** — replicas in different regions don't need a round-trip to a distant leader for every operation

---

## 3. Common Refinements (Stronger Than "Pure" Eventual)

|Model|Guarantee|
|---|---|
|**Read-Your-Writes**|A client always sees its own prior writes, even if other clients haven't yet (e.g. route a user's reads to the replica that handled their last write, or use sticky sessions)|
|**Monotonic Reads**|Once a client has seen a value, it never sees an older value on a subsequent read (prevents a value from "going back in time" from that client's perspective)|
|**Monotonic Writes**|A client's writes are applied in the order it issued them|
|**Causal Consistency**|Causally related operations (a reply depends on the original post) are seen in the same order by everyone; concurrent unrelated operations may be seen in different orders|

These sit between pure eventual consistency and strong consistency — most production "eventually consistent" systems actually implement one or more of these refinements rather than offering zero guarantees.

---

## 4. Conflict Resolution — What Happens When Replicas Disagree

Because writes can happen concurrently on different replicas before they've synced, conflicts are inevitable. Common strategies:

- **Last-Write-Wins (LWW)** — attach a timestamp, highest timestamp wins. Simple, but can silently drop a legitimate concurrent write (and depends on clock accuracy)
- **Vector Clocks** — track causality per-replica to detect _true_ concurrent writes (as opposed to one write happening-before another) and surface conflicts explicitly instead of silently picking one
- **CRDTs (Conflict-free Replicated Data Types)** — data structures (counters, sets, maps) specifically designed so concurrent updates can always be merged deterministically without conflict, no manual resolution needed (used in collaborative editing, distributed counters)
- **Application-level / manual resolution** — surface the conflict to the application or user (classic example: Amazon's old shopping cart merge — union of items from both concurrent cart versions rather than picking one)

---

## 5. Propagation Mechanisms

- **Gossip protocols** — nodes periodically exchange state with a few random peers, spreading updates across the cluster over several rounds (used by Cassandra, DynamoDB-style systems)
- **Read repair** — when a read notices replicas disagree, it triggers a background write to bring the stale replica up to date
- **Anti-entropy / Merkle trees** — background comparison of replica state (via hash trees) to efficiently find and fix divergence without comparing every single record

---

## 6. Where It Shows Up in Real Systems

| System                                              | Where eventual consistency is used                                                                                                           |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| [[DNS]]                                             | Propagation across resolvers can take time (TTL-bound) — classic, everyone has felt this                                                     |
| [[CDN]] / caches                                    | Edge nodes may briefly serve stale content after an origin update                                                                            |
| DynamoDB / Cassandra                                | Default consistency level for reads is eventual (tunable to stronger per-request)                                                            |
| Social feeds / like counts                          | Approximate counts are fine; users don't need perfect real-time agreement                                                                    |
| [[event-driven-architecture]] / [[kafka]] consumers | Consumers process events asynchronously — the "system" as a whole is eventually consistent even if individual writes are strongly consistent |
| DNS-based service discovery                         | Same propagation-delay trade-off as DNS generally                                                                                            |

---

## 7. When NOT to Use It

- Financial balances, inventory counts that must never go negative, anything requiring a single source of truth at the moment of decision → use strong consistency ([[consistency]]) or [[distributed-locks]] for the critical section
- Don't apply it by default just because it's "the scalable choice" — the right question is whether the _business_ can tolerate temporary staleness for that specific piece of data, not whether the system can technically support it

---

## 8. Interview Talking Points

- Always pair the claim "eventual consistency is fine here" with _why_, tied to the specific data (e.g. "a stale like count for a few seconds doesn't hurt anyone, but a stale account balance would") — interviewers are testing judgment, not just recall
- Bring up one concrete conflict-resolution mechanism (LWW is the easy one; CRDTs signal deeper knowledge) rather than hand-waving "conflicts get resolved"
- Mention read-your-writes as the specific refinement that fixes the most common user-facing complaint about eventual consistency ("I just posted this, why can't I see it?")
- Tie back to CAP/PACELC in [[consistency]] — eventual consistency is the explicit trade of C for A/latency, not an accident

---

## 9. Resources

- [Werner Vogels (Amazon CTO) – Eventually Consistent](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html)
- [Martin Kleppmann – Designing Data-Intensive Applications, Ch. 5 (Replication)](https://dataintensive.net/)
- [CRDT.tech – Conflict-free Replicated Data Types](https://crdt.tech/)

---

## Related Notes

- [[consistency]]
- [[distributed-locks]]
- [[fault-tolerance]]
- [[event-driven-architecture]]
- [[kafka]]
- [[reliability]]