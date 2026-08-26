# Isolation Levels (PostgreSQL & MongoDB)

## 1. Definition

Isolation level = how strictly a database prevents concurrent transactions from interfering with each other. It's a **trade-off dial**: stricter isolation = more correctness guarantees but more blocking/lower concurrency; weaker isolation = more throughput but more anomalies possible.

Prerequisite reading: [[transactions]] (concurrency problems: dirty read, non-repeatable read, phantom read, write skew).

---

## 2. The Standard SQL Isolation Levels

|Level|Dirty Read|Non-Repeatable Read|Phantom Read|Write Skew|
|---|---|---|---|---|
|**Read Uncommitted**|Possible|Possible|Possible|Possible|
|**Read Committed**|Prevented|Possible|Possible|Possible|
|**Repeatable Read**|Prevented|Prevented|Possible (SQL standard) / Prevented (PostgreSQL)|Possible|
|**Serializable**|Prevented|Prevented|Prevented|Prevented|

Weakest → strongest, top to bottom. Higher isolation = more correctness, less concurrency/throughput.

---

## 3. PostgreSQL Isolation Levels (MVCC-based)

PostgreSQL implements isolation using **MVCC** (Multi-Version Concurrency Control) — readers never block writers and vice versa, because each transaction sees a consistent **snapshot** of the data.

|PostgreSQL Level|Default?|Behavior|
|---|---|---|
|**Read Committed**|✅ Default|Each statement (not the whole transaction) sees a fresh snapshot as of when _that statement_ started. Most permissive; good default for most apps.|
|**Repeatable Read**|No|Entire transaction sees one snapshot, taken at the start of the transaction. PostgreSQL's Repeatable Read actually **prevents phantom reads too** (stronger than the SQL standard requires) via MVCC snapshot isolation.|
|**Serializable**|No|Full serializability via **SSI (Serializable Snapshot Isolation)** — detects and aborts transactions whose concurrent execution _would_ produce a non-serializable result, rather than using heavy locking.|

**Key PostgreSQL-specific fact for interviews:** PostgreSQL never actually implements literal "Read Uncommitted" — it's aliased to Read Committed, since dirty reads simply aren't possible under its MVCC model.

**Write skew example (still possible at Repeatable Read):**

```
Two doctors on-call; rule: at least 1 must remain on-call.
T1: reads {DrA on-call, DrB on-call} → decides to remove DrA
T2: reads {DrA on-call, DrB on-call} → decides to remove DrB
Both commit → 0 doctors on-call → invariant violated
```

Only **Serializable** isolation prevents this — it's the classic example used to justify Serializable in real systems.

---

## 4. MongoDB Isolation & Consistency Model

MongoDB's model differs because it's historically **document-oriented** with a different consistency story:

|Concept|MongoDB behavior|
|---|---|
|**Single-document atomicity**|Always atomic, even for nested fields/arrays within one document — no isolation level needed for this|
|**Multi-document transactions** (4.0+ replica sets, 4.2+ sharded clusters)|Supports **snapshot isolation** — a transaction sees a consistent snapshot across all documents/collections it touches|
|**Read Concern levels**|`local`, `available`, `majority`, `linearizable`, `snapshot` — control what data a read can see|
|**Write Concern levels**|`w: 1`, `w: majority`, etc. — control how many replicas must acknowledge a write before it's considered successful|

### MongoDB Read Concern Levels

|Read Concern|Guarantee|
|---|---|
|`local` (default)|Returns most recent data on this node — may be rolled back later if node was on the losing side of an election|
|`available`|Similar to local, used in sharded clusters, may show orphaned docs during migrations|
|`majority`|Only returns data acknowledged by a majority of replica set members — won't be rolled back|
|`linearizable`|Strongest — guarantees the read reflects all previously acknowledged majority-committed writes|
|`snapshot`|Used within multi-document transactions — consistent point-in-time view|

**Interview-relevant nuance:** MongoDB's "isolation" story is really a **combination** of read concern + write concern + transaction usage, not a single dial like SQL's isolation levels. Getting strong consistency in MongoDB means explicitly choosing `majority` read concern + `majority` write concern (or full transactions), whereas the default settings favor availability/speed — a direct CP/AP trade-off (see [[cap-theorem]]).

---

## 5. PostgreSQL vs MongoDB Isolation — Side by Side

| |PostgreSQL|MongoDB|
|---|---|---|
|Default consistency|Read Committed (single-node, always consistent within transaction)|`local` read concern — may read stale/rollback-able data on default settings|
|Multi-record transactions|Native, always available|Available since 4.0, but discouraged for high-throughput paths (perf cost)|
|Concurrency control|MVCC|MVCC-like (WiredTiger storage engine) + read/write concern tuning|
|Strongest guarantee|`SERIALIZABLE` (SSI)|`linearizable` read concern + `majority` write concern|
|Philosophy|ACID-first, single-node strong consistency by default|Tunable consistency — pick your trade-off per query/transaction|

---

## 6. How to Choose an Isolation Level (Practical Guidance)

1. **Start with the default** (Read Committed in Postgres) — it's sufficient for the vast majority of application logic.
2. **Use Repeatable Read** when a transaction needs a stable view across multiple reads (e.g. generating a report within one transaction).
3. **Use Serializable** only for genuinely conflict-sensitive logic (e.g. the doctors on-call example, inventory decrement race conditions) — it costs throughput (more aborts/retries under contention).
4. In MongoDB, **default read/write concerns are fine for most reads**; reach for `majority`/transactions specifically for financial/inventory-style operations where correctness matters more than latency.

---

## 7. Interview Talking Points

- Know that **Repeatable Read in PostgreSQL ≠ Repeatable Read in the SQL standard** — Postgres's snapshot isolation already blocks phantom reads, a good "I actually know this DB" signal.
- Explain **write skew** with a concrete example (on-call doctors, or double-booking a meeting room) — it's the most commonly asked "what can Repeatable Read NOT catch" question.
- For MongoDB questions, don't just say "eventually consistent" — mention **read concern / write concern as the actual tunable knobs**, and that `majority`+`majority` gets you strong-consistency-like guarantees.

---

## 8. Resources

- [PostgreSQL Docs – Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [MongoDB Docs – Read Concern](https://www.mongodb.com/docs/manual/reference/read-concern/)
- [MongoDB Docs – Transactions](https://www.mongodb.com/docs/manual/core/transactions/)
- _Designing Data-Intensive Applications_ by Martin Kleppmann — Ch. 7 (Weak Isolation Levels)
- [Jepsen – MongoDB analysis reports](https://jepsen.io/analyses) (real-world consistency testing)

---

## Related Notes

- [[transactions]]
- [[acid]]
- [[consistency]]
- [[cap-theorem]]
- [[databases]]