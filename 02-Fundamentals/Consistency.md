# Consistency

## 1. Definition

Consistency = whether **all clients/nodes see the same data at the same time**, particularly after a write.

It answers: _"If I write data on node A, and immediately read from node B, do I see the new value or the old one?"_

Two related but distinct meanings:

- **ACID consistency** (database transactions): the DB moves from one valid state to another, respecting constraints/invariants.
- **Distributed systems consistency** (CAP's "C"): all replicas agree on the current value of data. This note focuses mainly on the latter.

---

## 2. The Consistency Spectrum

From strongest (slowest, most coordination) to weakest (fastest, least coordination):

```
Strong  ────────────────────────────────────────►  Weak
Linearizable → Sequential → Causal → Read-your-writes → Eventual
```

|Model|Guarantee|Example use case|
|---|---|---|
|**Strong / Linearizable**|Every read sees the latest write, globally ordered, as if there were only one copy of data|Bank balance, inventory count|
|**Sequential**|All nodes see operations in the _same_ order, but not necessarily real-time|Distributed logs|
|**Causal consistency**|Causally-related operations are seen in order by everyone; unrelated ops may be seen in different orders|Comment threads (reply must appear after original)|
|**Read-your-writes**|A user always sees their _own_ writes immediately, but may see others' writes with delay|Social media profile edits (you see your own change instantly)|
|**Eventual consistency**|Given no new writes, all replicas _eventually_ converge to the same value — no timing guarantee|DNS, S3 (historically), shopping cart item counts|

---

## 3. Strong vs Eventual Consistency — Trade-offs

| |Strong Consistency|Eventual Consistency|
|---|---|---|
|**Correctness**|Always up to date|May be stale temporarily|
|**Latency**|Higher (needs coordination/quorum)|Lower (no waiting for consensus)|
|**Availability during partition**|Lower (per CAP, CP systems reject requests)|Higher (AP systems keep serving)|
|**Complexity**|Simpler to reason about|Requires conflict resolution logic|
|**Example systems**|Spanner, Zookeeper, traditional RDBMS|Cassandra, DynamoDB (default), DNS|

This directly maps to [[cap-theorem]]'s CP vs AP choice.

---

## 4. How Strong Consistency Is Achieved

|Mechanism|How it works|
|---|---|
|**Quorum reads/writes**|Require `W + R > N` acknowledgments (N = replicas) to guarantee overlap between read and write sets|
|**Consensus protocols**|Paxos, Raft — nodes agree on a single value/order despite failures|
|**Leader-based replication**|All writes go through a single leader, which orders them|
|**Two-phase commit (2PC)**|Coordinator ensures all nodes commit or none do (atomic distributed transaction)|
|**Distributed locks**|Prevent concurrent conflicting writes (e.g. via Zookeeper/etcd)|

---

## 5. Conflict Resolution in Eventually Consistent Systems

When two writes happen concurrently on different replicas, something must resolve the conflict when they sync:

|Strategy|How it works|
|---|---|
|**Last-write-wins (LWW)**|Timestamp-based; simplest, but can silently lose data|
|**Vector clocks**|Track causal history per replica to detect true conflicts vs ordered updates|
|**CRDTs** (Conflict-free Replicated Data Types)|Data structures designed to merge automatically without conflicts (e.g. counters, sets)|
|**Application-level merge**|e.g. shopping cart: merge = union of items instead of picking one version|

---

## 6. Read-Your-Writes & Session Consistency (Practical Middle Ground)

Many production systems don't need full strong consistency — they need the **user to see their own actions immediately**, while tolerating slight staleness for other users' actions. Implemented via:

- Sticky sessions (route user's reads to the replica that handled their write)
- Read-after-write from primary for a short window post-write

---

## 7. Interview Talking Points

- Always ask: "does this feature need strong consistency, or is eventual consistency acceptable?" — most features (feeds, view counts, recommendations) don't need strong consistency; financial/inventory operations usually do.
- Mention **quorum math** if asked about tunable consistency (e.g. Cassandra: `QUORUM` reads + `QUORUM` writes = strong consistency; `ONE` + `ONE` = fast but weaker guarantees).
- Connect back to [[cap-theorem]] and [[latency]] — stronger consistency generally costs more latency due to coordination overhead.

---

## 8. Resources

- _Designing Data-Intensive Applications_ by Martin Kleppmann — Ch. 5 (Replication) & Ch. 9 (Consistency and Consensus) — the best single resource on this topic
- [Jepsen – Consistency Models](https://jepsen.io/consistency)
- [AWS – Eventual vs Strong Consistency in DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)
- [CRDTs explained – Figma Engineering Blog](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/)

---

## Related Notes

- [[cap-theorem]]
- [[reliability]]
- [[availability]]
- [[system-design-networking-index]]