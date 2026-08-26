# CAP Theorem

## 1. Definition

In a **distributed system**, when a **network partition** occurs, you can only guarantee **two out of three** of:

- **C — Consistency**: every read receives the most recent write (or an error)
- **A — Availability**: every request receives a (non-error) response, without guaranteeing it's the latest data
- **P — Partition Tolerance**: the system continues to operate despite network failures/message loss between nodes

```
        Consistency
           /  \
          /    \
         CP    CA   (CA is theoretical only —
         /       \    real networks always partition)
        /         \
   Partition --- Availability
    Tolerance
```

**Key clarification (commonly misunderstood):** CAP is not "pick 2 of 3" in general — it's specifically about behavior **during a network partition**. Since partitions are _inevitable_ in real distributed systems, the real choice is: **when a partition happens, do you sacrifice Consistency or Availability?** P isn't really optional — it's a given.

---

## 2. Why "CA" Doesn't Really Exist

A single-node database is trivially CA (no partition possible). But any _distributed_ system must tolerate partitions — network failures will happen. So in practice, distributed systems are either:

- **CP** (Consistent + Partition-tolerant): during a partition, sacrifice availability — reject requests / block until consistency can be guaranteed
- **AP** (Available + Partition-tolerant): during a partition, sacrifice consistency — keep responding, but data may be stale/conflicting until resolved

---

## 3. CP vs AP — Examples

|System|Choice|Behavior during partition|
|---|---|---|
|**MongoDB** (default)|CP|Writes to a partitioned-away primary fail until a new primary is elected|
|**HBase**|CP|Regions unavailable until reassigned|
|**Zookeeper**|CP|Loses quorum → stops serving writes|
|**Cassandra**|AP (tunable)|Keeps accepting reads/writes on all reachable nodes; resolves conflicts later|
|**DynamoDB**|AP (tunable)|Eventually consistent by default, strong consistency optional per-request|
|**Riak**|AP|Always available, uses vector clocks to reconcile|

**Interview framing:** "Would you rather show a user slightly stale data, or show them an error?" — that's the CP/AP decision in plain English.

---

## 4. Real-World Example

**Banking system (CP-leaning):** Transferring money — you'd rather the transfer fail/retry than let two partitioned nodes both allow a withdrawal (double-spend risk). Consistency > Availability here.

**Social media like counter (AP-leaning):** If the like count is off by a few during a partition, no real harm. Availability > strict consistency — users should always be able to interact.

---

## 5. PACELC — The Extension (mention this to stand out in interviews)

CAP only describes partition-time behavior. **PACELC** adds the _no-partition_ case:

> **If Partition (P), choose between Availability (A) and Consistency (C); Else (E), choose between Latency (L) and Consistency (C).**

|System|Partition behavior|Normal-operation trade-off|
|---|---|---|
|DynamoDB|AP|EL (favors low Latency)|
|MongoDB|CP|EC (favors Consistency)|
|Cassandra|AP (tunable)|EL (tunable)|

This is a stronger, more complete mental model than CAP alone — CAP is necessary but not sufficient in interviews.

---

## 6. Consistency Models (relates to [[consistency]])

CAP's "C" is about **linearizability** (strong consistency), but real systems offer a spectrum:

- Strong consistency
- Eventual consistency
- Causal consistency
- Read-your-writes consistency

See [[consistency]] for the full breakdown.

---

## 7. Common Interview Mistakes to Avoid

- ❌ Saying "we chose CA" — doesn't exist for real distributed systems.
- ❌ Treating CAP as a permanent global system property — it's actually a per-operation/per-partition trade-off; many systems are _tunable_ (e.g. Cassandra lets you choose consistency level per query).
- ❌ Forgetting to mention PACELC when discussing latency trade-offs during normal operation.
- ✅ Do say: "During a partition we'd sacrifice X because [business reason]," tied to the actual use case.

---

## 8. Resources

- [Julia Evans – "You can't sacrifice partition tolerance"](https://jvns.ca/blog/2018/08/24/cap-theorem/) (accessible explainer)
- [Original CAP proof – Gilbert & Lynch, 2002 (PDF)](https://users.ece.cmu.edu/~adrian/731-sp04/readings/GL-cap.pdf)
- [PACELC – Daniel Abadi's original post](https://dbmsmusings.blogspot.com/2010/04/problems-with-cap-and-yahoos-little.html)
- [MongoDB Docs – Replica Set Elections (CP behavior example)](https://www.mongodb.com/docs/manual/core/replica-set-elections/)

---

## Related Notes

- [[consistency]]
- [[availability]]
- [[reliability]]
- [[system-design-networking-index]]