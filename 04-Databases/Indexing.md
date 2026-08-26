# Database Indexing

## 1. Definition

An index is a **separate data structure** that lets the database find rows without scanning the entire table — trading extra storage + slower writes for much faster reads on the indexed column(s).

Without an index, a query like `WHERE email = 'x@y.com'` requires a **full table scan** — checking every row.

---

## 2. B-Tree Index (Most Common)

The default index type in PostgreSQL, MySQL, MongoDB (WiredTiger), and most RDBMS.

```
                [50]
              /      \
          [20,35]    [70,90]
          /  |  \      /  |  \
        ... ... ...  ... ... ...
```

- Balanced tree — `O(log n)` lookups
- Great for **equality** (`=`) and **range queries** (`<`, `>`, `BETWEEN`)
- Keeps data sorted, so also speeds up `ORDER BY` on the indexed column

---

## 3. Other Index Types

|Type|Best for|Example|
|---|---|---|
|**Hash Index**|Pure equality lookups only (no ranges)|`WHERE id = 5`|
|**B-Tree**|Equality + range queries, sorting|Default choice for most columns|
|**Full-text index** (inverted index)|Text search|`WHERE content MATCHES 'search term'`|
|**Geospatial index** (R-tree, geohash)|Location-based queries|"find restaurants within 5km"|
|**Bitmap index**|Low-cardinality columns (few distinct values), analytics workloads|`WHERE status = 'active'`|
|**Composite (multi-column) index**|Queries filtering/sorting on multiple columns together|`WHERE last_name = ? AND first_name = ?`|

---

## 4. How an Inverted Index Works (Full-Text Search)

Maps **terms → list of documents/rows containing them** (reverse of document → terms):

```
"quick"  → [doc1, doc5, doc9]
"brown"  → [doc1, doc3]
"fox"    → [doc1, doc9]
```

Used by Elasticsearch, PostgreSQL's `tsvector`, Lucene. Querying "quick fox" becomes an intersection of postings lists, which is fast.

---

## 5. Composite Indexes & Column Order Matters

An index on `(last_name, first_name)` is **not** the same as `(first_name, last_name)`:

- It efficiently supports queries filtering on `last_name` alone, OR `last_name + first_name` together
- It does **NOT** efficiently support filtering on `first_name` alone (like a phone book sorted by last name — you can't binary-search it by first name)

**Rule of thumb (interview-worthy):** order composite index columns by **equality filters first, then range filters, then sort columns** — this maximizes how much of the index the query planner can use.

---

## 6. Covering Index

An index that contains **all columns** a query needs — the database can answer the query entirely from the index without touching the actual table ("index-only scan"). Faster because it avoids an extra disk read per row.

```sql
-- Query: SELECT user_id, status FROM orders WHERE user_id = 5
CREATE INDEX idx_covering ON orders (user_id, status);
-- fully "covers" the query above
```

---

## 7. The Cost of Indexes (Trade-off — Don't Over-Index)

|Benefit|Cost|
|---|---|
|Much faster reads on indexed columns|Slower writes (every `INSERT`/`UPDATE`/`DELETE` must also update each index)|
|Faster sorting/grouping on indexed columns|More storage (each index is a separate structure, sometimes as large as the table itself)|
||More work for the query planner to choose the right index|

**Interview point:** indexing everything is a common junior mistake — always tie index decisions to actual **query patterns** (what does `WHERE`/`JOIN`/`ORDER BY` actually filter/sort on?), not just "index every column."

---

## 8. How the Query Planner Decides (PostgreSQL example)

Run `EXPLAIN ANALYZE <query>` to see whether the planner used:

- **Sequential Scan** — full table scan (index not used — often means missing index, or the table is small enough that scanning is actually cheaper)
- **Index Scan** — uses the index, then fetches matching rows from the table
- **Index-Only Scan** — covering index, no table access needed (fastest)
- **Bitmap Index Scan** — used when combining multiple indexes or when many rows match (more efficient than repeated random I/O)

---

## 9. Indexing in MongoDB

Conceptually similar (B-Tree under the hood), with additional index types:

- **Single field, compound, multikey** (indexing array fields), **text**, **geospatial**, **hashed** (for sharding, see [[sharding]])
- `explain()` is MongoDB's equivalent of `EXPLAIN ANALYZE`
- **ESR rule** for compound indexes: order fields as **E**quality, **S**ort, **R**ange — MongoDB's version of the composite index ordering rule above

---

## 10. Interview Talking Points

- Always tie an index recommendation to a **specific query pattern** you're optimizing — never suggest indexing in the abstract.
- Mention the **write cost trade-off** explicitly — shows you understand indexes aren't free.
- Bring up **composite index column ordering** (equality → range → sort) when discussing multi-column filters — a strong differentiator in interviews.
- For search-heavy features, distinguish "just add a B-Tree index" from "you actually need a full-text/inverted index (or a dedicated search engine like Elasticsearch)."

---

## 11. Resources

- [Use The Index, Luke! (free, excellent, SQL-focused)](https://use-the-index-luke.com/)
- [PostgreSQL Docs – Indexes](https://www.postgresql.org/docs/current/indexes.html)
- [MongoDB Docs – Indexes](https://www.mongodb.com/docs/manual/indexes/)
- _Designing Data-Intensive Applications_ by Martin Kleppmann — Ch. 3 (Storage and Retrieval)

---

## Related Notes

- [[databases]]
- [[sharding]]
- [[latency]]
- [[sql-vs-nosql]]