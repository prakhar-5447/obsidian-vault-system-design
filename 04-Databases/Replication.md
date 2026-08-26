# Database Replication

## 1. Definition

Replication = keeping **copies of the same data on multiple nodes**, to improve [[availability]] (survive node failure), [[scalability]] (spread read load), and durability.

Not to be confused with [[sharding]] — replication copies the **same** data to multiple nodes; sharding splits **different** data across multiple nodes. Systems often use both together.

---

## 2. Replication Topologies

### Single-Leader (Primary-Replica / Master-Slave)

```
        Writes
          ↓
      [ Leader ]
       /    \
  [Replica] [Replica]   ← Reads (optionally)
```

- All writes go to one **leader**; **replicas** apply the leader's write log to stay in sync
- Reads can be served from replicas to scale read throughput
- Simple to reason about — no write conflicts (single writer)
- **Failover** required if the leader dies (promote a replica) — adds complexity + brief unavailability

**Used by:** PostgreSQL (streaming replication), MySQL, MongoDB (replica sets — the primary IS this pattern)

### Multi-Leader (Multi-Master)

```
[Leader A] <──sync──> [Leader B]
    ↑                      ↑
 Writes (region 1)     Writes (region 2)
```

- Multiple nodes accept writes independently, then sync with each other
- Useful for **multi-region** setups (each region writes locally, low latency)
- **Conflict resolution required** — two leaders can accept conflicting writes to the same record before syncing (last-write-wins, vector clocks, CRDTs — see [[consistency]])

**Used by:** CouchDB, some MySQL multi-master setups, geographically distributed systems

### Leaderless (Peer-to-Peer)

```
[Node A] ↔ [Node B] ↔ [Node C]
   any node accepts reads/writes
```

- **Any** node can accept a write; client (or coordinator) writes to multiple nodes directly
- Uses **quorum** reads/writes (`W + R > N`) to maintain consistency guarantees despite no single leader
- Highly available — no single point of failure, no failover process needed

**Used by:** Cassandra, DynamoDB, Riak

---

## 3. Synchronous vs Asynchronous Replication

| |Synchronous|Asynchronous|
|---|---|---|
|Leader waits for replica ACK before confirming write?|Yes|No|
|Durability|Stronger — write is guaranteed on ≥2 nodes before client gets "success"|Weaker — leader could crash before replica catches up, losing the write|
|Latency|Higher (waits for replica RTT)|Lower (doesn't wait)|
|Availability|Lower (if replica is down, writes may stall)|Higher (leader never blocked by replica)|

**Semi-synchronous** (common middle ground): wait for **at least one** replica to ack (not all) — balances durability and latency. This is a direct instance of the [[latency]] vs [[consistency]]/durability trade-off, and connects to [[cap-theorem]]/[[availability]] math.

---

## 4. Replication Lag & Its Consequences

Async replicas can fall behind the leader — this is **replication lag**, and it causes real, interview-relevant problems:

|Problem|Scenario|
|---|---|
|**Read-after-write inconsistency**|User posts a comment (written to leader), immediately refreshes and reads from a lagging replica → doesn't see their own comment|
|**Monotonic read violation**|User reads from replica A (sees new data), then reads from replica B (lagging further) → sees _older_ data than before, looks like time went backward|

**Mitigations:**

- **Read-your-writes**: route a user's reads to the leader (or a replica known to be caught up) for a short window after their own write
- **Sticky sessions**: pin a user's reads to the same replica for a session, avoiding the "different replica, different lag" problem
- **Monitor replication lag** as an operational metric — alert if it exceeds a threshold

---

## 5. How Replication Actually Propagates Changes

|Method|How it works|
|---|---|
|**Statement-based**|Replicate the SQL statement itself, re-execute on replica|
|**Write-ahead log (WAL) shipping**|Ship the low-level WAL bytes; replica replays them|
|**Logical/row-based replication**|Replicate the actual row changes (not the statement or raw WAL)|
|**Change Data Capture (CDC)**|Stream row-level changes to external systems (Kafka, Debezium)|

---

## 6. Failover Process (Single-Leader Systems)

1. Detect leader failure (health checks / heartbeat timeout)
2. Elect a new leader (most up-to-date replica, often via consensus like Raft)
3. Reconfigure clients/DNS/proxy to point to new leader
4. Old leader (if it comes back) must recognize it's no longer leader (avoid **split-brain** — two nodes both believing they're the leader, causing conflicting writes)

**Split-brain** is a serious failure mode — mitigated via consensus protocols (Raft) or fencing mechanisms (e.g. STONITH — "Shoot The Other Node In The Head").

---

## 7. Interview Talking Points

- Always specify **sync vs async** when describing a replication design — it directly determines your durability/latency/availability trade-off, and interviewers expect you to know this dial exists.
- Bring up **replication lag** and its user-facing symptoms (stale reads) — a strong signal you understand real production behavior, not just the textbook diagram.
- Mention **CDC** when discussing how to keep a search index, cache, or analytics store in sync with the primary database — a very common real system-design pattern.
- Connect explicitly to [[cap-theorem]]: sync replication leans CP-ish (blocks on partition), async leans AP-ish (keeps serving, risks staleness).

---

## 8. Resources

- _Designing Data-Intensive Applications_ by Martin Kleppmann — Ch. 5 (Replication) — the definitive resource
- [PostgreSQL Docs – Streaming Replication](https://www.postgresql.org/docs/current/warm-standby.html)
- [MongoDB Docs – Replica Sets](https://www.mongodb.com/docs/manual/replication/)
- [Debezium (CDC) Docs](https://debezium.io/documentation/)

---

## Related Notes

- [[sharding]]
- [[availability]]
- [[consistency]]
- [[cap-theorem]]
- [[databases]]