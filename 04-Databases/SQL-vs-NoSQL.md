# SQL vs NoSQL

## 1. Definition

**SQL (relational)** databases store data in **tables with fixed schemas**, related via foreign keys, queried with SQL. **NoSQL** is an umbrella term for non-relational databases — document, key-value, column-family, and graph stores — generally optimized for flexibility and horizontal scale over strict schema/relations.

"NoSQL" doesn't mean "no rules" — it means "not only SQL," and each NoSQL category has its own strengths.

---

## 2. NoSQL Categories

|Type|Data model|Example DBs|Best for|
|---|---|---|---|
|**Document**|JSON-like documents, nested fields|MongoDB, Couchbase|Flexible/evolving schemas, content management, catalogs|
|**Key-Value**|Simple key → value|Redis, DynamoDB, Memcached|Caching, session storage, simple lookups|
|**Column-family (wide-column)**|Rows with dynamic columns, grouped by column families|Cassandra, HBase, BigTable|Write-heavy, time-series, large-scale analytics|
|**Graph**|Nodes + edges, optimized for relationship traversal|Neo4j, Amazon Neptune|Social networks, recommendation engines, fraud detection|

---

## 3. SQL vs NoSQL — Core Comparison

| |SQL (Relational)|NoSQL|
|---|---|---|
|Schema|Fixed, defined upfront (migrations needed to change)|Flexible/dynamic (document fields can vary per record)|
|Relationships|Native (foreign keys, JOINs)|Usually denormalized/embedded; joins are app-level or limited|
|Transactions|Strong ACID, multi-row/table by default|Historically single-document/row atomic only; multi-doc transactions now available in some (MongoDB 4.0+) but at a cost|
|Consistency|Strong by default (single-node)|Often tunable — eventual by default in distributed setups (see [[consistency]])|
|Scaling|Primarily vertical; horizontal via sharding is harder (needs tooling like Vitess/Citus)|Built for horizontal scaling from the ground up (native sharding, leaderless replication)|
|Query flexibility|Powerful ad-hoc queries (JOINs, aggregations, subqueries)|Varies — document DBs support rich queries; key-value stores are lookup-only|
|Best for|Structured data with clear relationships, strong consistency needs (finance, inventory)|High-scale, flexible/evolving schema, high write throughput, denormalized access patterns|

---

## 4. Why NoSQL Scales Horizontally More Easily

- **No cross-row/table JOINs to coordinate** across nodes — most NoSQL query patterns are single-key lookups, which shard trivially
- **Relaxed consistency** — many NoSQL systems default to eventual consistency, avoiding the coordination overhead that strong consistency requires across a distributed cluster (ties to [[cap-theorem]])
- **Schema flexibility** avoids expensive schema migrations that get harder at scale in SQL systems (e.g. `ALTER TABLE` on a billion-row table can lock/take a long time)

This is a _design philosophy_ difference, not a strict technical limitation — modern systems like **CockroachDB, Google Spanner, YugabyteDB, Vitess, Citus** provide SQL semantics with horizontal scalability, blurring the line. Worth mentioning in interviews as "NewSQL."

---

## 5. Data Modeling Difference — Normalization vs Denormalization

**SQL (normalized):**

```
users table:        orders table:
id | name            id | user_id | item
1  | Prakhar          1  | 1       | Book
                       2  | 1       | Pen
```

Query needs a JOIN to get a user's orders.

**NoSQL / MongoDB (denormalized, embedded):**

```json
{
  "user_id": 1,
  "name": "Prakhar",
  "orders": [
    { "item": "Book" },
    { "item": "Pen" }
  ]
}
```

One read gets everything — no JOIN needed, but data is duplicated if orders are also queried independently, and updates to shared/repeated data must be applied in multiple places.

**Rule of thumb:** in NoSQL, **model around your query patterns**, not around normalized entity relationships — "denormalize for reads" is the core document-DB design principle.

---

## 6. When to Choose SQL

- Data has clear, stable relationships (users, orders, payments) that benefit from JOINs and referential integrity
- Strong consistency and multi-row ACID transactions matter (banking, inventory, anything involving money)
- Complex ad-hoc querying/reporting is a core requirement
- Schema is relatively stable and well-understood upfront

## 7. When to Choose NoSQL

- Schema is evolving/unpredictable (rapid product iteration, varied document shapes)
- Massive write throughput or horizontal scale requirements from day one (IoT, logging, event streams)
- Access pattern is mostly key-based lookups, not complex relational queries
- Need flexible, nested/hierarchical data (product catalogs with wildly different attributes per category)

---

## 8. Polyglot Persistence (Realistic Answer for System Design Interviews)

Most real large-scale systems **don't pick just one** — they use different databases for different subsystems based on each subsystem's access pattern:

|Subsystem|Likely choice|Why|
|---|---|---|
|User accounts, payments|PostgreSQL/MySQL|Strong consistency, ACID transactions|
|Product catalog|MongoDB|Flexible, varied schema per product type|
|Session/cache layer|Redis|Fast key-value, in-memory|
|Search|Elasticsearch|Full-text/inverted index|
|Analytics/logs|Cassandra / column-store (e.g. ClickHouse)|Write-heavy, time-series friendly|
|Social graph/recommendations|Neo4j|Relationship traversal|

**This is the strongest interview answer** to "SQL or NoSQL?" — reason about each subsystem's actual access pattern rather than picking one database for the whole system.

---

## 9. Interview Talking Points

- Never answer "SQL vs NoSQL" as a binary preference — walk through the **access pattern, consistency need, and scale requirement** for the specific subsystem being discussed (ties to [[databases]] decision framework).
- Mention **NewSQL** (CockroachDB, Spanner) to show awareness that "SQL doesn't scale horizontally" is an outdated generalization.
- Bring up **denormalization** as the key document-DB modeling shift — a common practical question ("how would you model this in MongoDB?").

---

## 10. Resources

- _Designing Data-Intensive Applications_ by Martin Kleppmann — Ch. 2 (Data Models and Query Languages)
- [MongoDB – Data Modeling Guide](https://www.mongodb.com/docs/manual/core/data-modeling-introduction/)
- [Google Spanner Paper (NewSQL horizontal scale + ACID)](https://research.google/pubs/pub39966/)
- [Martin Fowler – Polyglot Persistence](https://martinfowler.com/bliki/PolyglotPersistence.html)

---

## Related Notes

- [[databases]]
- [[acid]]
- [[sharding]]
- [[consistency]]
- [[cap-theorem]]