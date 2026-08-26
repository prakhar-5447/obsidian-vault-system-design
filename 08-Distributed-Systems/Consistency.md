# Consistency (Distributed Systems)

## 1. Definition

Consistency describes **what guarantees a distributed system makes about the order and visibility of data changes** across replicas/nodes. It's one of the three properties in the **CAP theorem** (Consistency, Availability, Partition tolerance) — during a network partition, a system must choose between staying consistent or staying available, it can't fully have both. See [[eventual-consistency]] for the weakest common relaxation of strict consistency.

---

## 2. CAP Theorem Refresher

| |Meaning|
|---|---|
|**Consistency (C)**|Every read receives the most recent write (or an error) — all nodes see the same data at the same time|
|**Availability (A)**|Every request receives a (non-error) response, without guarantee it's the most recent write|
|**Partition tolerance (P)**|The system continues to operate despite network partitions between nodes|

**In practice: partition tolerance is not optional** in a real distributed system (networks _will_ partition) — so the real trade-off is **C vs A during a partition**, giving rise to **CP systems** (e.g. [[distributed-locks|ZooKeeper]], most consensus-based DBs) and **AP systems** (e.g. Cassandra, DynamoDB by default). This is often called "CAP is really about CP vs AP."

**PACELC extends this**: even without a Partition, there's a trade-off between **Latency and Consistency (ELC)** — stronger consistency generally requires more coordination, which costs latency.

---

## 3. Consistency Models (Strongest → Weakest)

|Model|Guarantee|Cost|
|---|---|---|
|**Strict/Linearizable Consistency**|Every read reflects the latest write, as if there were a single copy of the data, with real-time ordering|Highest coordination cost, hurts availability/latency under partition|
|**Sequential Consistency**|All operations appear in _some_ consistent global order (matching each process's own program order), but not necessarily real-time order|Slightly weaker than linearizability, still requires global agreement|
|**Causal Consistency**|Operations that are causally related (e.g. a reply to a comment) are seen by everyone in the same order; unrelated (concurrent) operations may be seen in different orders on different nodes|Good balance — preserves the ordering that actually matters to users, without full global coordination|
|**Read-Your-Writes Consistency**|A user always sees their own writes, even if other users haven't yet|Common in practice (e.g. route a user's reads to the primary/same replica that handled their write)|
|**Eventual Consistency**|Given no new writes, all replicas _eventually_ converge to the same value — no bound on when|Highest availability/lowest latency, weakest guarantee — see [[eventual-consistency]]|

---

## 4. Strong Consistency — How It's Achieved

- **Single leader / primary** — all writes go through one node, reads from the leader are always up to date (reads from followers may lag)
- **Consensus protocols** (Raft, Paxos) — a write is only committed once a **majority (quorum)** of nodes agree, so any subsequent read from a majority is guaranteed to see it
- **Quorum reads/writes** — with N replicas, require W nodes to ack a write and R nodes to agree on a read, where **W + R > N** guarantees overlap between the write set and read set, giving strong consistency without requiring all N nodes

---

## 5. Consistency vs the Rest of the System

| Trade-off partner            | Relationship                                                                                                                                      |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **[[Availability]]**         | CAP — stronger consistency generally reduces availability during a partition                                                                      |
| **[[Latency]]**              | PACELC — stronger consistency requires more coordination round-trips                                                                              |
| **[[eventual-consistency]]** | The explicit choice to relax consistency in exchange for availability/latency/throughput                                                          |
| **[[distributed-locks]]**    | Distributed locking is one mechanism used to enforce strong consistency for a critical section across nodes                                       |
| **[[fault-tolerance]]**      | Consensus-based strong consistency (Raft/Paxos) is itself a fault-tolerance mechanism — it tolerates minority node failures while staying correct |

---

## 6. Practical Guidance: Picking a Model

- Financial transactions, inventory counts, locks/leader election → **strong consistency** (correctness matters more than latency)
- Social media feeds, view counts, "likes", product recommendations → **eventual consistency** is usually fine and much cheaper
- Most real systems are **mixed** — e.g. strongly consistent for the order/payment path, eventually consistent for the recommendation/analytics path

---

## 7. Interview Talking Points

- Reframe CAP correctly if the interviewer asks it as a simple 3-way pick — the real-world framing is **CP vs AP under partition**, and mentioning PACELC (latency trade-off even without a partition) shows deeper understanding
- Name the specific consistency model the design needs, not just "consistent" or "eventually consistent" — e.g. "this needs read-your-writes, not full linearizability" is a strong, precise answer
- Bring up quorum reads/writes (W + R > N) as the concrete mechanism when asked how to tune the consistency/availability dial
- Connect to specific databases when relevant: DynamoDB/Cassandra (tunable, AP-leaning by default), Spanner/CockroachDB (strong, via consensus + synchronized clocks), traditional single-leader RDBMS (strong by default, single point of coordination)

---

## 8. Resources

- [Jepsen – Consistency Models](https://jepsen.io/consistency)
- [Martin Kleppmann – Designing Data-Intensive Applications, Ch. 9 (Consistency & Consensus)](https://dataintensive.net/)
- [AWS – PACELC and CAP explained](https://aws.amazon.com/builders-library/)

---

## Related Notes

- [[eventual-consistency]]
- [[distributed-locks]]
- [[fault-tolerance]]
- [[reliability]]