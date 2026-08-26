# Transactions

## 1. Definition

A transaction is a **group of one or more operations** (reads/writes) executed as a **single logical unit of work** — either all of it takes effect, or none of it does. Governed by [[acid]] guarantees.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

If any statement fails before `COMMIT`, you `ROLLBACK` and neither update persists.

---

## 2. Why Transactions Exist

Without transactions, a crash or concurrent access mid-operation can leave data in a **partially updated, inconsistent state** — e.g. money debited from one account but never credited to another. Transactions provide the "all or nothing" guarantee that makes multi-step operations safe.

---

## 3. Transaction Lifecycle

```
BEGIN → [operations] → COMMIT (persist permanently)
                     → ROLLBACK (undo everything)
```

- **Commit**: changes become permanent and visible to other transactions
- **Rollback**: all changes in the transaction are discarded, as if it never ran
- **Savepoints**: partial rollback points within a larger transaction (rollback to a specific point instead of the whole transaction)

---

## 4. Concurrency Problems Transactions Must Handle

|Problem|Description|
|---|---|
|**Dirty read**|Reading uncommitted data from another transaction that might later roll back|
|**Non-repeatable read**|Reading the same row twice in one transaction gives different results (another transaction committed a change in between)|
|**Phantom read**|Re-running the same query returns a _different set of rows_ (another transaction inserted/deleted matching rows)|
|**Lost update**|Two transactions read-modify-write the same value; one update silently overwrites the other|
|**Write skew**|Two transactions each read overlapping data, make decisions based on it, and write — individually valid, but combined violate an invariant|

These are exactly what **isolation levels** are designed to prevent (to varying degrees) — full breakdown in [[isolation-levels]].

---

## 5. Concurrency Control Mechanisms

|Mechanism|How it works|
|---|---|
|**Pessimistic locking (2PL — Two-Phase Locking)**|Acquire locks before reading/writing; hold until commit. Prevents conflicts but can cause blocking/deadlocks.|
|**Optimistic Concurrency Control (OCC)**|Don't lock; check for conflicts at commit time (e.g. version number mismatch) and abort/retry if conflict detected. Better for low-contention workloads.|
|**MVCC (Multi-Version Concurrency Control)**|Keep multiple versions of a row; readers see a consistent snapshot without blocking writers, and vice versa. Used by PostgreSQL, MySQL InnoDB, MongoDB (WiredTiger).|

**MVCC is the dominant modern approach** — it lets reads never block writes and vice versa, which is why PostgreSQL/MongoDB perform well under mixed read/write load without heavy lock contention.

---

## 6. Deadlocks

Two transactions each hold a lock the other needs → both wait forever.

```
T1: locks Row A, wants Row B
T2: locks Row B, wants Row A
→ deadlock
```

**Detection/resolution:** databases run a deadlock detector (wait-for graph) and **abort one transaction** (the "victim") to break the cycle, which then retries. Application code should always be ready to retry on deadlock errors.

**Prevention tip (interview-relevant):** always acquire locks on multiple resources in a **consistent order** across your application (e.g. always lock lower ID first) to avoid deadlock cycles entirely.

---

## 7. Distributed Transactions

When a transaction spans multiple nodes/shards/services, single-node mechanisms (WAL, local locks) aren't enough — see [[acid]] section 5-6 for **Two-Phase Commit** and the **Saga pattern**.

---

## 8. Transactions in NoSQL

Historically, NoSQL databases only guaranteed atomicity at the **single-document/single-row** level, explicitly trading multi-record transactions for scalability. This has evolved:

- **MongoDB** (4.0+) supports multi-document ACID transactions, but they carry a performance cost and are recommended only when genuinely needed.
- **DynamoDB** supports transactions across multiple items within certain limits.
- **Cassandra** offers lightweight transactions (LWT) via Paxos-based conditional writes, not full ACID.

See [[sql-vs-nosql]] for the broader trade-off context.

---

## 9. Interview Talking Points

- When designing anything involving money/inventory, explicitly call out **where transaction boundaries are** and **why** (e.g. "reserve inventory and charge payment must be atomic, or in a distributed context, use a Saga with compensation").
- Mention **MVCC** when asked how a DB handles concurrent reads/writes without blocking — a strong signal of real understanding.
- Bring up **lock ordering** as a concrete deadlock-prevention technique if asked about concurrency bugs.

---

## 10. Resources

- _Designing Data-Intensive Applications_ by Martin Kleppmann — Ch. 7 (Transactions) — best single resource
- [PostgreSQL Docs – Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [MongoDB Docs – Transactions](https://www.mongodb.com/docs/manual/core/transactions/)
- [Use The Index, Luke! – Locking basics](https://use-the-index-luke.com/)

---

## Related Notes

- [[acid]]
- [[isolation-levels]]
- [[databases]]
- [[reliability]]