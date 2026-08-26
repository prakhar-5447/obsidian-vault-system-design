
# Horizontal vs Vertical Scaling

## 1. Definition

Two fundamentally different ways to give a system more capacity:

- **Vertical scaling (scale up)** — add more resources (CPU, RAM, disk, faster network) to a **single existing machine**
- **Horizontal scaling (scale out)** — add **more machines**, and distribute load across all of them

This is the foundational decision underlying [[scalability]] — worth its own note since interviewers frequently probe this trade-off directly.

---

## 2. Side-by-Side Comparison

| |Vertical Scaling|Horizontal Scaling|
|---|---|---|
|**How**|Bigger instance (more CPU/RAM/disk)|More instances (replicas/nodes)|
|**Ceiling**|Hard limit — largest available machine|Practically unlimited — keep adding nodes|
|**Downtime to scale**|Often required (resize = reboot/migration)|None — add nodes without touching existing ones|
|**Complexity added**|Minimal — same architecture, bigger box|Significant — load balancing, data partitioning, distributed coordination|
|**Single point of failure**|Yes (still one machine)|No — inherently redundant across nodes|
|**Cost curve**|Non-linear — high-end hardware is disproportionately expensive|More linear — commodity hardware, but more instances to pay for/manage|
|**Code changes needed**|Usually none|Often significant (statelessness required, data partitioning logic, distributed transactions)|
|**Good fit for**|Databases with strong consistency needs, moderate/predictable load, simpler systems|Web/app tiers, high/unpredictable load, systems needing high availability|

---

## 3. Why Vertical Scaling Hits a Ceiling

- Physical hardware limits (max CPU cores, max RAM per machine) — even the largest cloud instances (e.g. AWS's largest EC2 types) cap out
- Cost grows **non-linearly** — doubling CPU/RAM on a high-end machine often costs far more than double
- **Single point of failure remains** no matter how big the machine — one big box going down still takes everything with it, unaddressed by vertical scaling alone (ties to [[availability]])

---

## 4. Why Horizontal Scaling Is Harder Than It Sounds

Adding more machines only helps if the **workload can actually be distributed** — this requires:

|Requirement|Why|
|---|---|
|**Statelessness**|A server holding local session/data can't be freely load-balanced — state must live externally (Redis, DB) so any instance can handle any request|
|**Data partitioning** (for databases)|A single-writer database doesn't get faster writes just by adding read replicas — see [[sharding]]|
|**Coordination overhead**|Multiple nodes now need to agree on shared state (consensus, locks) — this is where [[cap-theorem]] and [[consistency]] trade-offs enter|
|**Load balancing**|Traffic must actually be spread evenly across the new nodes — see [[load-balancing]]|
|**Network overhead**|Inter-node communication (that didn't exist on one machine) adds latency — see [[latency]]|

This is why horizontal scaling is a **design decision made early**, not a switch flipped later — retrofitting statelessness and partitioning onto an architecture built assuming a single machine is often a major rewrite.

---

## 5. What Scales Well Horizontally vs What Doesn't

|Scales horizontally easily|Harder to scale horizontally|
|---|---|
|Stateless web/API servers|Single-writer relational databases (without sharding tooling like Citus)|
|Read replicas (for read-heavy DB load)|Systems requiring strong global ordering/consistency|
|Stateless workers/consumers (message queue consumers)|Stateful in-memory session data (unless externalized)|
|CDN/edge caching|Anything requiring frequent cross-node coordination|
|Microservices with independent data stores|Monolithic apps with a single shared DB and tight coupling|

---

## 6. Practical Guidance — Which to Use When

1. **Start vertical** — it's simpler, requires no architecture changes, and is often sufficient until real scale is reached. Most systems never need more than a well-sized single database instance for the primary write path.
2. **Move to horizontal for the stateless tiers first** — web/app servers are the easiest, lowest-risk place to start horizontal scaling (just add instances behind a load balancer).
3. **Only shard the database when genuinely necessary** — read replicas + caching often defer this need significantly; sharding (see [[sharding]]) is the last resort due to its complexity cost.
4. **Combine both in practice** — real systems vertically scale individual nodes to a reasonable size _and_ horizontally scale the number of those nodes, rather than picking one exclusively.

---

## 7. Interview Talking Points

- Don't default to "just scale horizontally" for every component — explicitly reason about **which tier** needs it and why (stateless app tier vs stateful database tier have very different stories).
- Mention that horizontal scaling requires **statelessness as a precondition** — a strong, frequently-tested detail.
- When discussing database scaling specifically, walk through the **progression**: vertical scaling → read replicas → caching → sharding, rather than jumping straight to "shard it."
- Tie the horizontal scaling discussion back to [[availability]] — redundancy as a side benefit of horizontal scaling, not just a throughput lever.

---

## 8. Resources

- _Designing Data-Intensive Applications_ by Martin Kleppmann — Ch. 1 (Scalability)
- [AWS – Scaling Overview](https://aws.amazon.com/what-is/vertical-scaling/)
- [System Design Primer – Scalability section](https://github.com/donnemartin/system-design-primer#scalability)

---

## Related Notes

- [[scalability]]
- [[availability]]
- [[sharding]]
- [[load-balancing]]
- [[cap-theorem]]