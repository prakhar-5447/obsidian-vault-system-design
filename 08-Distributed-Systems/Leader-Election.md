# Leader Election

## 1. Definition

Leader election is the process by which a group of distributed nodes **agree on a single node** to act as the coordinator/primary for some responsibility — e.g. accepting writes ([[replication]]'s single-leader model), scheduling a job exactly once, or coordinating a distributed task — even though nodes can fail or the network can partition at any time.

---

## 2. Why It's Needed

Many distributed responsibilities require **exactly one** active coordinator at a time:

- A database replica set needs one primary accepting writes (see [[replication]])
- A cron-like scheduled job running across multiple app instances should fire **once**, not once per instance
- A distributed lock service needs to know who currently holds a lock

Without leader election, you either need manual configuration (fragile, doesn't survive failure) or risk **split-brain** — two nodes both believing they're the leader simultaneously, causing conflicting writes/actions.

---

## 3. Core Requirements of a Correct Leader Election Algorithm

- **Safety**: at most one leader is active at any time (no split-brain)
- **Liveness**: if the current leader fails, a new one is eventually elected (system doesn't get stuck leaderless forever)
- **Fault tolerance**: survives the failure of a minority of nodes

These map directly onto **consensus** — leader election is a specific application of the more general distributed consensus problem.

---

## 4. Consensus Algorithms Behind Leader Election

### Raft (Most Widely Used Today — Designed for Understandability)

Nodes are in one of three states: **Follower**, **Candidate**, **Leader**.

```
All nodes start as Followers
   ↓ (no heartbeat from leader within timeout)
Follower becomes Candidate → requests votes from other nodes
   ↓ (receives majority of votes)
Candidate becomes Leader → sends periodic heartbeats to prevent new elections
```

- Uses **randomized election timeouts** so candidates don't all trigger elections simultaneously (reduces split-vote likelihood)
- Requires a **majority (quorum)** of nodes to elect a leader — this is why clusters are typically sized in **odd numbers** (3, 5, 7) — a majority is well-defined and tolerates `(N-1)/2` node failures
- **Used by:** etcd, Consul, CockroachDB, Kafka (KRaft mode, replacing ZooKeeper)

### Paxos

Older, more theoretically foundational consensus algorithm — mathematically equivalent in power to Raft but notoriously harder to understand and implement correctly (part of why Raft was created — explicitly designed to be more approachable). **Used by:** Google Chubby, Google Spanner (via a Paxos variant).

### ZooKeeper's ZAB (ZooKeeper Atomic Broadcast)

ZooKeeper's own consensus protocol, similar in spirit to Raft/Paxos, used historically as the **coordination service** many other distributed systems relied on for leader election (Kafka used to depend on ZooKeeper for this before moving to KRaft/Raft-based self-management).

---

## 5. Quorum — The Core Mechanism

A **quorum** is the minimum number of nodes that must agree for a decision (like electing a leader) to be valid — typically `⌊N/2⌋ + 1` (a strict majority).

**Why majority, and why odd N:**

```
N = 3 nodes → quorum = 2 → tolerates 1 failure
N = 5 nodes → quorum = 3 → tolerates 2 failures
N = 4 nodes → quorum = 3 → tolerates only 1 failure (worse ratio than N=3, wastes a node)
```

Even-numbered clusters don't help — they cost more nodes without improving fault tolerance, and can risk **split votes** in an election (two candidates each get exactly half). This is why odd cluster sizes (3, 5, 7) are the standard recommendation.

---

## 6. Leader Election with Coordination Services (Practical Implementation)

Most real systems don't hand-roll Raft — they use an existing coordination service:

|Service|How leader election works|
|---|---|
|**ZooKeeper**|Nodes create sequential ephemeral znodes; the node with the lowest sequence number is the leader; if it dies, its znode disappears (ephemeral), and the next-lowest node takes over|
|**etcd**|Nodes attempt to acquire a lease-backed key; whoever holds it is the leader; lease auto-expires if the leader stops renewing (crashes), triggering re-election|
|**Consul**|Similar session/lock-based leader election primitive built on Raft internally|

**Ephemeral nodes/leases are the key trick:** the "who is leader" state is tied to an active, renewed connection/lease — if the leader crashes, the coordination service detects the lost connection/expired lease and automatically triggers a new election, without needing the other nodes to explicitly detect the failure themselves.

---

## 7. Failure Scenarios to Reason About

|Scenario|What should happen|
|---|---|
|**Leader crashes**|Followers detect missing heartbeats, trigger election, new leader chosen within a bounded time|
|**Network partition splits cluster in two**|Only the side with a **majority** can elect/keep a leader; the minority side cannot elect one (prevents split-brain) — directly a CP choice per [[cap-theorem]]|
|**Leader is alive but network-partitioned from followers**|Followers (correctly) elect a new leader after timeout; old leader must recognize it's lost quorum and step down (**fencing**) to avoid two active leaders|
|**Split vote** (candidates tie)|Randomized timeouts (Raft) make repeated ties statistically unlikely; election simply retries|

**Fencing** is critical: a leader that loses connectivity but doesn't realize it (or comes back after being replaced) must be prevented from continuing to act as leader — often via a **fencing token** (monotonically increasing number) that downstream systems check, rejecting writes from a stale/outdated leader.

---

## 8. Interview Talking Points

- Bring up leader election whenever a design needs **exactly one** coordinator for something (single writer, scheduled job dedup, distributed lock) — a strong, precise signal versus vaguely saying "we'll coordinate somehow."
- Mention **quorum and odd-numbered clusters** specifically — a well-known, checkable detail.
- Reference **Raft** by name as the modern default choice (used by etcd, Consul, CockroachDB, newer Kafka) rather than Paxos, unless discussing older/Google-specific systems.
- Bring up **fencing tokens** if asked about a stale leader coming back online — shows you understand that election alone doesn't fully solve split-brain without an enforcement mechanism downstream.
- Tie back explicitly to [[cap-theorem]]: requiring a majority to elect/act is a CP choice — the minority partition sacrifices availability to preserve safety.

---

## 9. Resources

- [Raft Consensus Algorithm – The Secret Lives of Data (interactive visualization)](https://thesecretlivesofdata.com/raft/)
- [Raft Paper – "In Search of an Understandable Consensus Algorithm"](https://raft.github.io/raft.pdf)
- [ZooKeeper Docs – Leader Election Recipe](https://zookeeper.apache.org/doc/current/recipes.html#sc_leaderElection)
- [etcd Docs – Distributed Locking & Leader Election](https://etcd.io/docs/latest/learning/why/)
- _Designing Data-Intensive Applications_ by Martin Kleppmann — Ch. 9 (Consistency and Consensus)

---

## Related Notes

- [[replication]]
- [[cap-theorem]]
- [[consistency]]
- [[reliability]]