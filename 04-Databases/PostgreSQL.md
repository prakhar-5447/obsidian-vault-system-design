# PostgreSQL

## 1. Overview

PostgreSQL is an **open-source, object-relational database** known for strict standards compliance, extensibility, and strong ACID guarantees via MVCC. Widely considered the default "serious" choice among open-source RDBMSs for system design interviews. Prerequisite context: [[acid]], [[transactions]], [[isolation-levels]], [[indexing]].

---

## 2. Storage & MVCC Internals

- Every row update creates a **new row version** (tuple) rather than overwriting in place — old versions remain until cleaned up
- **VACUUM** — background process that reclaims space from dead tuples (old row versions no longer visible to any transaction); `VACUUM FULL` reclaims more aggressively but locks the table
- **Autovacuum** — runs automatically; under-tuned autovacuum on high-write tables is a very common real-world production issue (table bloat, degraded performance) — worth mentioning as an operational gotcha
- **WAL (Write-Ahead Log)** — every change is written to the WAL before being applied, enabling crash recovery and forming the basis of streaming replication (see [[replication]])

---

## 3. Indexing in PostgreSQL

Beyond the general concepts in [[indexing]], Postgres-specific index types:

|Index type|Use case|
|---|---|
|**B-Tree** (default)|Equality + range queries, sorting|
|**GIN** (Generalized Inverted Index)|Full-text search, JSONB containment queries, array columns|
|**GiST**|Geometric data, full-text search, nearest-neighbor queries|
|**BRIN** (Block Range Index)|Very large tables with naturally sorted data (e.g. time-series by timestamp) — tiny index size, trades precision for space|
|**Hash**|Pure equality only, rarely used in practice since B-Tree covers this too|

**Partial indexes** — index only rows matching a condition, saving space/write cost:

```sql
CREATE INDEX idx_active_users ON users (email) WHERE is_active = true;
```

**Expression indexes** — index a computed expression, not just a raw column:

```sql
CREATE INDEX idx_lower_email ON users (LOWER(email));
```

---

## 4. JSONB — Postgres as a Semi-Structured Store

Postgres supports `JSONB` (binary JSON) columns — queryable, indexable (via GIN), letting you get some document-DB flexibility **inside** a relational database:

```sql
SELECT * FROM products WHERE attributes @> '{"color": "red"}';
CREATE INDEX idx_attrs ON products USING GIN (attributes);
```

**Interview point:** this is a real, common alternative to reaching for MongoDB when you mostly need relational structure but a _few_ columns are variable/schema-flexible — avoids running two databases just for that.

---

## 5. Replication & High Availability

- **Streaming replication** — WAL records shipped to replicas in near real-time (async by default, can be sync — see [[replication]])
- **Logical replication** — replicate specific tables/changes (not the whole WAL), enables selective sync, zero-downtime major version upgrades, and feeding CDC pipelines (Debezium)
- **Failover tooling**: Patroni, repmgr — Postgres doesn't have fully built-in automatic failover like MongoDB replica sets; typically layered with external tooling in production

---

## 6. Scaling PostgreSQL

Postgres is **not natively horizontally sharded** — this is the most common comparison point vs MongoDB/Cassandra:

|Approach|How|
|---|---|
|**Vertical scaling**|Bigger instance — the simplest first lever|
|**Read replicas**|Scale reads; writes still bottlenecked at one primary|
|**Connection pooling** (PgBouncer)|Postgres connections are relatively expensive (process-per-connection model) — pooling is almost always needed at scale, a very commonly forgotten interview detail|
|**Citus** (extension)|Adds native sharding to Postgres — turns it into a distributed database while keeping SQL/Postgres compatibility|
|**Partitioning** (native, single-node)|Split one large table into smaller physical pieces (by range/list/hash) on the _same_ server — improves query performance and maintenance (e.g. dropping old partitions), but is not the same as sharding across multiple machines|

**Partitioning vs Sharding (important distinction):** partitioning splits data within a single node for manageability/performance; sharding (see [[sharding]]) splits data across multiple nodes for horizontal scale. Postgres partitioning is native and common; Postgres sharding needs an extension like Citus or app-level sharding.

---

## 7. Concurrency & Locking

- MVCC means readers never block writers and vice versa (see [[isolation-levels]])
- Explicit locking available (`SELECT ... FOR UPDATE`) for cases needing pessimistic concurrency control (see [[transactions]])
- `pg_locks` and `pg_stat_activity` — the go-to diagnostic views for debugging lock contention/long-running queries in production

---

## 8. When to Choose PostgreSQL (Interview Framing)

- Strong relational structure, need JOINs and referential integrity
- Need genuine multi-row ACID transactions
- Complex querying/reporting requirements (window functions, CTEs, full SQL power)
- Willing to trade native horizontal write-scaling for consistency/simplicity, or plan to add Citus/sharding later if truly needed
- Want JSONB for the _occasional_ flexible field without going fully document-oriented

---

## 9. Interview Talking Points

- Mention **connection pooling (PgBouncer)** whenever discussing Postgres at scale — a detail many candidates miss that signals real production experience.
- Clearly distinguish **partitioning** (single-node) from **sharding** (multi-node) — a very common conflation.
- Bring up **JSONB** as a middle-ground answer when asked "would you use Postgres or Mongo here" for a mostly-relational system with a bit of flexible data.
- Mention **autovacuum tuning** as a real operational concern for high-write tables — shows awareness beyond schema design.

---

## 10. Resources

- [PostgreSQL Official Docs](https://www.postgresql.org/docs/current/)
- [Use The Index, Luke!](https://use-the-index-luke.com/)
- [Citus Docs (sharding Postgres)](https://docs.citusdata.com/)
- [PgBouncer Docs](https://www.pgbouncer.org/)
- _PostgreSQL: Up and Running_ by Regina Obe & Leo Hsu — good practical book reference

---

## Related Notes

- [[databases]]
- [[acid]]
- [[isolation-levels]]
- [[indexing]]
- [[replication]]
- [[sharding]]
- [[sql-vs-nosql]]
- [[mongodb]]