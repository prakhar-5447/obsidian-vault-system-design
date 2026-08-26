# Database Sharding

## 1. Definition

Sharding (a.k.a. horizontal partitioning) = splitting a **single logical dataset** across **multiple database nodes**, where each node ("shard") holds a **subset** of the data. Done when data/write volume outgrows what one machine can handle.

Different from [[replication]]: sharding splits **different** data across nodes; replication copies **the same** data across nodes. Production systems combine both — e.g. 4 shards, each with 3 replicas.

---

## 2. Why Shard (When to Reach for This)

- Dataset too large to fit/perform well on one machine (storage, memory, index size)
- Write throughput exceeds what a single primary can handle (a hard limit — a single-leader system, per [[replication]], has ONE write node)
- Read replicas alone don't help if the bottleneck is **write** volume

**Interview rule:** don't reach for sharding prematurely — first exhaust vertical scaling, read replicas, and caching (see [[scalability]]). Sharding adds significant complexity (cross-shard queries, rebalancing, distributed transactions) and should be justified by an actual scale requirement.

---

## 3. Sharding Strategies

|Strategy|How it works|Pros|Cons|
|---|---|---|---|
|**Range-based**|Shard by key ranges (e.g. user_id 1-1M → shard 1, 1M-2M → shard 2)|Simple; efficient range queries within a shard|Hot shards if data/access isn't uniform (e.g. newest users all hit the last shard)|
|**Hash-based**|`hash(key) % num_shards` determines shard|Even data distribution|Range queries now require hitting _every_ shard; resharding is expensive (see consistent hashing below)|
|**Directory-based**|A lookup service maps each key → its shard|Very flexible (can rebalance by just updating the directory)|Directory service is an extra hop + potential bottleneck/SPOF|
|**Geo-based**|Shard by user's region/location|Low latency (data near user); helps data residency/compliance (GDPR etc.)|Uneven load if regions have very different traffic|

---

## 4. Choosing a Shard Key (The Most Important Decision)

The shard key determines how data is distributed — a bad choice causes **hot shards** (uneven load) despite having many shards.

|Good shard key traits|Bad shard key example|
|---|---|
|High cardinality (many distinct values)|`country` (few values, huge skew — e.g. most users in one country)|
|Evenly distributes read/write load|`created_at` timestamp (all _new_ writes hit the latest/single shard — "hot" shard)|
|Matches your most common query pattern (avoid cross-shard queries for hot paths)|An ID unrelated to how you actually query the data|

**Common good choice:** `user_id` (high cardinality, most queries are naturally scoped per-user) — but even this can create hot shards for "celebrity" users with disproportionate activity (see "hot key" problem below).

---

## 5. Consistent Hashing (Solves the Resharding Problem)

Naive `hash(key) % N` means **adding/removing a shard remaps almost all keys** — catastrophic for a live system (massive data movement).

**Consistent hashing** arranges shards on a conceptual **ring**; each key hashes to a point on the ring and is owned by the next shard clockwise. Adding/removing a shard only remaps the keys between it and its neighbor — a small fraction of total keys, not all of them.

```
        Shard A
       /        \
  Shard D        Shard B
       \        /
        Shard C

key → hash(key) → nearest shard clockwise on ring
```

**Virtual nodes**: each physical shard is mapped to multiple points on the ring, which smooths out load distribution (avoids one shard getting a disproportionate arc just by hash luck).

**Used by:** Cassandra, DynamoDB, Discord's data infra, many CDN/caching systems.

---

## 6. Challenges Sharding Introduces

|Challenge|Why it's hard|
|---|---|
|**Cross-shard queries/joins**|A query needing data from multiple shards (e.g. `JOIN` across shard boundaries) requires app-level fan-out + merge — no native distributed join in most sharded setups|
|**Cross-shard transactions**|Multi-shard atomic writes need 2PC or Sagas (see [[acid]]) — much harder than single-node transactions|
|**Rebalancing**|Adding/removing shards requires moving data — consistent hashing minimizes but doesn't eliminate this|
|**Hot shards / hot keys**|Uneven access patterns concentrate load on one shard despite "even" key distribution (e.g. a viral post's shard gets hammered)|
|**Operational complexity**|Backups, monitoring, schema migrations now must happen consistently across N shards instead of 1|

---

## 7. Application-Level Sharding vs Built-In Sharding

| |Application-level sharding|Built-in (native) sharding|
|---|---|---|
|Who decides which shard?|Your app code (often via a shard-key → shard-ID mapping)|The database itself (e.g. MongoDB sharded clusters, Vitess for MySQL, Citus for Postgres)|
|Complexity|You own routing logic, rebalancing, cross-shard queries|Database handles routing transparently via a query router|
|Flexibility|Full control|Easier to operate, less custom code, but tied to that DB's sharding model|

**MongoDB's native sharding**: uses a `mongos` query router + shard key + config servers tracking chunk ranges — you pick the shard key, MongoDB handles chunk splitting/balancing automatically.

---

## 8. Interview Talking Points

- Always justify sharding with an actual scale number (e.g. "at 50M users and 10K writes/sec, a single Postgres primary becomes the bottleneck") — don't jump to sharding by default.
- Explain **why naive `hash % N` is bad** and consistent hashing fixes it — a strong signal of depth.
- Proactively mention the **shard key choice trade-off** and hot-shard risk — this is the #1 follow-up question interviewers ask.
- Bring up how cross-shard queries/transactions are handled (scatter-gather query, Saga pattern) rather than ignoring the complexity sharding introduces.

---

## 9. Resources

- _Designing Data-Intensive Applications_ by Martin Kleppmann — Ch. 6 (Partitioning) — the definitive resource
- [MongoDB Docs – Sharding](https://www.mongodb.com/docs/manual/sharding/)
- [Vitess Docs (MySQL sharding at YouTube scale)](https://vitess.io/docs/)
- [Consistent Hashing – original paper summary (Karger et al.)](https://www.cs.princeton.edu/courses/archive/fall09/cos518/papers/chash.pdf)
- [Discord Engineering – How Discord Stores Trillions of Messages (real sharding case study)](https://discord.com/blog/how-discord-stores-trillions-of-messages)

---

## Related Notes

- [[replication]]
- [[scalability]]
- [[databases]]
- [[indexing]]