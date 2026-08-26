# Databases — Overview

## 1. What This Note Covers

A map of the database sub-topics in this vault, plus the high-level mental model for how they fit together. Deep dives live in their own notes: [[acid]], [[transactions]], [[isolation-levels]], [[indexing]], [[replication]], [[sharding]], [[sql-vs-nosql]].

---

## 2. Core Responsibilities of Any Database

|Responsibility|Related note|
|---|---|
|Store data durably|[[acid]] (Durability)|
|Let you query it efficiently|[[indexing]]|
|Keep it correct under concurrent access|[[transactions]], [[isolation-levels]]|
|Survive failure|[[replication]]|
|Scale beyond one machine|[[sharding]]|
|Pick the right data model|[[sql-vs-nosql]]|

---

## 3. Storage Engine Basics (Context for Indexing)

Most databases persist data using one of two dominant storage structures:

|Structure|Used by|Read/Write characteristics|
|---|---|---|
|**B-Tree**|PostgreSQL, MySQL (InnoDB), most RDBMS|Balanced read/write, in-place updates, good for range queries|
|**LSM-Tree** (Log-Structured Merge Tree)|Cassandra, RocksDB, LevelDB, MongoDB (WiredTiger uses a B-Tree variant, but many NoSQL stores use LSM)|Writes are fast (append-only, sequential), reads may need to check multiple levels; background compaction|

**Why it matters:** write-heavy systems (logging, time-series, event ingestion) often favor LSM-tree-based stores; read-heavy systems with complex queries favor B-Tree-based RDBMS. This single choice underlies a lot of the SQL vs NoSQL engine-level trade-off — see [[sql-vs-nosql]].

---

## 4. Database Design Decision Framework (Interview Use)

When asked "what database would you use for X," walk through:

1. **Access pattern** — read-heavy or write-heavy? Point lookups or range scans? Ad-hoc queries or fixed known queries?
2. **Consistency needs** — strong or eventual? (ties to [[consistency]], [[cap-theorem]])
3. **Data shape** — relational (joins, foreign keys) or document/key-value/graph friendly?
4. **Scale** — will it outgrow a single node? (ties to [[sharding]], [[scalability]])
5. **Transaction needs** — multi-record atomic operations required? (ties to [[transactions]], [[acid]])

---

## 5. Resources

- _Designing Data-Intensive Applications_ by Martin Kleppmann — the single best resource covering everything in this section, especially Part I & II
- [Use The Index, Luke! – free indexing deep dive](https://use-the-index-luke.com/)
- [PostgreSQL Official Docs](https://www.postgresql.org/docs/)
- [MongoDB Official Docs](https://www.mongodb.com/docs/)

---

## Related Notes

- [[acid]]
- [[transactions]]
- [[isolation-levels]]
- [[indexing]]
- [[replication]]
- [[sharding]]
- [[sql-vs-nosql]]
- [[system-design-networking-index]]