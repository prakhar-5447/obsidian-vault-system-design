# ACID

## 1. Definition

ACID is a set of four guarantees that a **database transaction** should provide to be considered reliable:

**A**tomicity, **C**onsistency, **I**solation, **D**urability.

Primarily associated with relational/SQL databases, though many NoSQL systems now offer ACID guarantees at least at the single-document/single-row level (and some, like FoundationDB or CockroachDB, offer full multi-record ACID transactions).

---

## 2. The Four Properties

### Atomicity

All operations in a transaction succeed, or **none** do — "all or nothing."

> Example: transferring money = debit account A + credit account B. If the credit fails after the debit succeeds, atomicity rolls back the debit too — you never end up with money vanishing.

### Consistency

A transaction moves the database from one **valid state to another**, respecting all defined constraints (foreign keys, unique constraints, triggers, application-defined invariants).

> Note: this is _not_ the same "C" as in [[cap-theorem]] — that C is about replica agreement in distributed systems. ACID's C is about respecting schema/business rule invariants within a single transaction.

### Isolation

Concurrent transactions shouldn't interfere with each other — each transaction behaves as if it were the only one running, to the degree specified by its **isolation level**. Full breakdown in [[isolation-levels]].

### Durability

Once a transaction is **committed**, it survives — even a crash immediately after commit won't lose the data (typically achieved via **write-ahead logging (WAL)**, flushed to disk before acknowledging commit).

---

## 3. How Each Property Is Actually Implemented

| Property    | Common implementation                                                                        |
| ----------- | -------------------------------------------------------------------------------------------- |
| Atomicity   | Write-ahead log (WAL) + rollback segments; undo uncommitted changes on failure               |
| Consistency | Constraint checks (foreign keys, unique indexes, `CHECK` constraints) enforced before commit |
| Isolation   | Locking (2PL) or MVCC (Multi-Version Concurrency Control) — see [[isolation-levels]]         |
| Durability  | WAL flushed + fsynced to disk before returning "commit success" to the client                |

---

## 4. ACID vs BASE (NoSQL Alternative Philosophy)

| ACID            | BASE                        |
| --------------- | --------------------------- |
| **A**tomicity   | **B**asically **A**vailable |
| **C**onsistency | **S**oft state              |
| **I**solation   | **E**ventual consistency    |
| **D**urability  |                             |

BASE trades strict correctness guarantees for **availability and scalability** — the philosophy behind most NoSQL systems (Cassandra, DynamoDB). This maps directly onto the CP/AP split in [[cap-theorem]]: ACID databases typically lean CP, BASE systems typically lean AP.

---

## 5. ACID in Distributed Systems (Why It Gets Hard)

Single-node ACID is well understood (WAL + locks/MVCC). **Distributed ACID** (transactions spanning multiple nodes/shards) requires coordination:

- **Two-Phase Commit (2PC)** — coordinator asks all participants to "prepare," then commits only if all say yes. Blocks on coordinator failure; not partition-tolerant.
- **Consensus-based approaches** (Raft/Paxos) — used by systems like CockroachDB, Google Spanner to achieve distributed ACID with better fault tolerance than 2PC.
- **Sagas** — an alternative pattern for _long-running_ distributed transactions that avoids holding locks across services by breaking a transaction into a sequence of local transactions with **compensating actions** for rollback (common in microservices — see below).

---

## 6. ACID in Microservices — The Saga Pattern

In a microservices architecture, a single business transaction (e.g. "place an order") often spans multiple services (inventory, payment, shipping) — no single database, so classic ACID transactions don't apply across services.

**Saga pattern:** break it into local transactions, each with a **compensating transaction** to undo it if a later step fails.

```
Reserve Inventory → Charge Payment → Create Shipment
       ↓ (if payment fails)
Compensate: Release Inventory
```

Two coordination styles: **choreography** (services react to each other's events, no central coordinator) vs **orchestration** (a central saga orchestrator drives the sequence).

---

## 7. Interview Talking Points

- Distinguish ACID's "Consistency" from CAP's "Consistency" explicitly if asked — a common trip-up.
- When asked about distributed transactions, mention the **Saga pattern** as the practical microservices answer instead of naive distributed 2PC (which doesn't scale/tolerate failure well).
- Tie durability directly to **replication** — durability guarantees are much stronger when WAL is also replicated to other nodes before ack (synchronous replication) vs written to a single disk.

---

## 8. Resources

- _Designing Data-Intensive Applications_ by Martin Kleppmann — Ch. 7 (Transactions)
- [Microservices.io – Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [PostgreSQL Docs – Reliability and the Write-Ahead Log](https://www.postgresql.org/docs/current/wal-intro.html)
- [Google Spanner Paper (distributed ACID at scale)](https://research.google/pubs/pub39966/)

---

## Related Notes

- [[transactions]]
- [[isolation-levels]]
- [[replication]]
- [[cap-theorem]]
- [[consistency]]
- [[databases]]