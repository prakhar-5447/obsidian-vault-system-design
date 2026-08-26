# Distributed Locks

## 1. Definition

A distributed lock ensures that **only one process (across multiple machines) can hold a lock and access a shared resource or critical section at a time** — the distributed-systems equivalent of a mutex, but harder because there's no shared memory and any node (or the network) can fail mid-operation. Used to enforce mutual exclusion for things like leader election, preventing duplicate job processing, or serializing access to a shared resource. Closely tied to [[consistency]] — a correct lock is fundamentally a consensus problem.

---

## 2. Why It's Harder Than a Local Lock

- **No shared memory** — a local mutex relies on the OS/language runtime; a distributed lock needs an external coordinator all nodes trust
- **Clock/timing unreliability** — you can't assume synchronized clocks or bounded message delays across nodes (see the classic "GC pause / network delay causes a client to think it still holds the lock when it doesn't" failure mode)
- **Partial failures** — the lock holder can crash while holding the lock, requiring some form of automatic expiry (a lease) rather than relying on the holder to release it
- **Network partitions** — a client might be unable to reach the lock service but still be alive and still think it holds the lock, and continue acting on that (false) assumption — this is the dangerous case, not just "the lock takes longer"

---

## 3. Common Implementation Approaches

### Redis (SET NX + TTL)

```
SET lock:resource_id client_id NX EX 30
```

- `NX` — only set if the key doesn't exist (atomic acquire)
- `EX 30` — auto-expire after 30s (a **lease**, not a permanent lock) so a crashed holder doesn't block others forever
- Release: only delete if the value matches your own `client_id` (via a Lua script, to make check-and-delete atomic — otherwise you might delete a lock someone else acquired after your lease expired)

**Redlock** — Redis's proposed algorithm for distributed locking across multiple independent Redis nodes (acquire on a majority to tolerate a single node failure). It's **controversial** — Martin Kleppmann published a well-known critique arguing Redlock doesn't provide the safety guarantees it claims under certain failure scenarios (clock drift, GC pauses), while Redis's creator (antirez) published a rebuttal. Worth knowing this debate exists, not just the mechanism, for a senior-level interview.

### ZooKeeper / etcd (Consensus-Based)

- Built on Raft/ZAB consensus — inherently more rigorous than a single Redis instance
- ZooKeeper: create an **ephemeral sequential znode**; the client with the lowest sequence number holds the lock; if a client dies, its session (and znode) is automatically cleaned up
- etcd: similar pattern using lease-bound keys and its own consensus-backed guarantees
- Generally considered **safer** than single-instance Redis locks for correctness-critical use cases, at the cost of more operational complexity and higher latency per lock operation

### Database-Based (Advisory Locks / Unique Constraints)

- Postgres advisory locks (`pg_advisory_lock`), or simply a unique constraint / row-level lock on a "locks" table
- Simple if you already depend on the DB, but ties lock availability to DB availability and doesn't scale to high lock-churn workloads

---

## 4. The Fencing Token Problem

The core danger: a client acquires a lock, then experiences a long pause (GC, network delay) during which its lease **expires** and another client acquires the lock — but the first client, unaware its lease expired, resumes and still writes to the shared resource, now **concurrently with the second lock holder**. The lock alone didn't guarantee mutual exclusion in practice.

**Fix: fencing tokens** — the lock service hands out a monotonically increasing number with each lock grant. The protected resource (e.g. storage service) rejects any write tagged with a token lower than the highest it's already seen. This pushes the correctness check to the resource itself rather than trusting the lock holder's belief that it still holds the lock.

---

## 5. Lease Duration Trade-off

|Shorter lease|Longer lease|
|---|---|
|Faster recovery if holder crashes|Slower recovery if holder crashes|
|Higher risk of premature expiry under load/GC pauses → false "lock lost"|Lower risk of premature expiry|
|Requires the holder to renew (heartbeat) more frequently|Less renewal overhead|

Most production systems implement **lease renewal (heartbeating)** — the holder periodically extends its lease while still working, rather than picking one fixed duration and hoping the work finishes in time.

---

## 6. Interview Talking Points

- Don't present a distributed lock as trivially solved by `SET NX` — bring up the fencing token problem unprompted, it's the detail that separates a shallow answer from a strong one
- Be ready to compare Redis (fast, simpler, weaker guarantees, good for best-effort deduplication) vs ZooKeeper/etcd (slower, consensus-backed, better for correctness-critical locking like leader election)
- Mention the Redlock debate if asked to defend a Redis-based approach at scale — shows awareness that "it's in a blog post" isn't the same as "it's proven correct"
- Connect back to [[consistency]]: a distributed lock is really asking for linearizable agreement on "who holds this right now," which is why it's built on the same consensus primitives (Raft/ZAB) as strongly consistent systems

---

## 7. Resources

- [Martin Kleppmann – How to do distributed locking (critique of Redlock)](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [Redis – Redlock algorithm](https://redis.io/docs/latest/develop/use/patterns/distributed-locks/)
- [ZooKeeper – Locks Recipe](https://zookeeper.apache.org/doc/current/recipes.html#sc_recipes_Locks)
- [etcd – Distributed Locks](https://etcd.io/docs/v3.5/dev-guide/api_concurrency_reference_v3/)

---

## Related Notes

- [[consistency]]
- [[eventual-consistency]]
- [[fault-tolerance]]
- [[reliability]]