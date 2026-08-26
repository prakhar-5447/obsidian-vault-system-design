# MongoDB

## 1. Overview

MongoDB is a **document-oriented NoSQL database** — stores data as BSON (binary JSON) documents in collections, with a flexible schema. Built from the ground up for horizontal scale and developer-friendly iteration speed. Prerequisite context: [[sql-vs-nosql]], [[isolation-levels]] (MongoDB-specific section already covers read/write concerns in depth), [[sharding]].

---

## 2. Document Data Model

```json
{
  "_id": ObjectId("..."),
  "name": "Prakhar",
  "orders": [
    { "item": "Book", "price": 12.99 },
    { "item": "Pen", "price": 1.50 }
  ]
}
```

- Documents in the same collection **can have different fields** — schema is enforced by convention/application code, not the database (unless you opt into **schema validation**, see below)
- Related data is often **embedded** (nested) rather than joined — see [[sql-vs-nosql]] section 5 for the normalization vs denormalization trade-off this implies

---

## 3. Embedding vs Referencing (Core Modeling Decision)

| |Embed|Reference (like a foreign key)|
|---|---|---|
|How|Nest related data directly in the document|Store an `_id` and query separately (or `$lookup`, Mongo's JOIN-like op)|
|Best for|Data always accessed together, doesn't grow unbounded (e.g. an address inside a user doc)|Data accessed independently, or that grows large/unbounded (e.g. a user's thousands of orders)|
|Risk if misused|Documents exceeding the **16MB document size limit**; duplicated data if the same sub-object appears in many places and needs updating everywhere|Extra round-trips (`$lookup` is slower than a native SQL JOIN, and less optimized)|

**Rule of thumb (frequently asked in interviews):** embed for **"contains"** relationships with bounded size and read-together access patterns; reference for **"references"** relationships, especially one-to-many where "many" can grow large, or many-to-many relationships.

---

## 4. Schema Validation (Optional, Not Schema-less by Necessity)

MongoDB isn't forced to be schema-less — you can enforce structure via JSON Schema validation rules on a collection:

```javascript
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      required: ["name", "email"],
      properties: {
        email: { bsonType: "string", pattern: "^.+@.+$" }
      }
    }
  }
})
```

**Interview point:** "MongoDB has no schema" is a common oversimplification — production MongoDB deployments very often use schema validation once the data model stabilizes, gaining safety while keeping flexibility for genuinely variable fields.

---

## 5. Indexing

Covered in depth in [[indexing]] section 9 — B-Tree based, supports compound, multikey (array fields), text, geospatial, and hashed indexes. **ESR rule** (Equality, Sort, Range) governs compound index field ordering, the Mongo equivalent of SQL's composite index ordering rule.

`explain("executionStats")` is the MongoDB equivalent of SQL's `EXPLAIN ANALYZE` — always mention checking this before/after adding an index in an interview.

---

## 6. Replication — Replica Sets

MongoDB's replication is **built-in and automatic** (unlike Postgres, which needs external tooling like Patroni for automated failover — see [[postgresql]]):

```
Primary (accepts writes)
   ↕ (oplog replication)
Secondary, Secondary, ... (can serve reads if configured)
```

- **Oplog** (operations log) — capped collection recording all writes, replicated to secondaries (Mongo's equivalent of a WAL)
- **Automatic election** — if the primary fails, remaining members (need a majority) elect a new primary automatically, typically within seconds
- Minimum recommended: **3 members** (odd number, avoids split-vote ties) — general concept covered in [[replication]] section 6 (failover/split-brain avoidance)

---

## 7. Sharding — Native, Built-In

Unlike Postgres (needs Citus extension), MongoDB has **native sharding**:

```
Client → mongos (query router) → determines shard via shard key → routes to correct shard
                                        ↑
                              Config servers track chunk metadata
```

- **Shard key** choice is the critical decision — same hot-shard risks discussed generally in [[sharding]] section 4
- **Chunks** — ranges of shard-key values; MongoDB automatically splits and rebalances chunks across shards as data grows
- **Hashed sharding** vs **range sharding** — same trade-offs as the general sharding note (hashed = even distribution but poor range queries; range = good range queries but hot-shard risk)

---

## 8. Read/Write Concerns & Transactions

Already covered in depth in [[isolation-levels]] sections 4-6 — the short version:

- **Read concern** (`local`, `majority`, `linearizable`) controls what a read is allowed to see
- **Write concern** (`w: 1`, `w: majority`) controls how many replicas must ack a write
- **Multi-document ACID transactions** available since 4.0 (replica sets) / 4.2 (sharded clusters) — powerful but carries a performance cost; use for genuinely multi-document atomic needs (e.g. a transfer between two documents), not by default for every write

---

## 9. Aggregation Framework

MongoDB's equivalent of SQL's `GROUP BY`/JOINs/window functions — a **pipeline** of stages:

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$userId", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } }
])
```

Worth mentioning by name in interviews when asked how MongoDB handles analytics-style queries without native SQL.

---

## 10. When to Choose MongoDB (Interview Framing)

- Schema is evolving or naturally variable across records (product catalogs with different attributes per category)
- Need to scale horizontally from early on, with built-in sharding/replication rather than bolted-on tooling
- Access pattern is mostly "fetch one document with everything I need" (denormalized reads) rather than complex multi-table JOINs
- Team wants faster iteration without schema migrations blocking every change

---

## 11. Interview Talking Points

- Lead with the **embed vs reference** decision when asked to model data in MongoDB — the single most common follow-up question.
- Mention that **schema validation exists** — avoids the "schema-less = no safety" misconception.
- Bring up **native sharding + automatic replica set failover** as MongoDB's key operational advantage over vanilla Postgres, while being honest that Postgres + Citus/Patroni can match much of this with more setup.
- When discussing consistency, reference the **read/write concern** knobs specifically (see [[isolation-levels]]) rather than a vague "eventual consistency" answer.

---

## 12. Resources

- [MongoDB Official Docs](https://www.mongodb.com/docs/manual/)
- [MongoDB University (free courses)](https://learn.mongodb.com/)
- [MongoDB – Data Modeling Patterns](https://www.mongodb.com/docs/manual/core/data-model-design/)
- [MongoDB – Sharding Docs](https://www.mongodb.com/docs/manual/sharding/)

---

## Related Notes

- [[databases]]
- [[sql-vs-nosql]]
- [[isolation-levels]]
- [[indexing]]
- [[replication]]
- [[sharding]]
- [[postgresql]]