# Availability

## 1. Definition

Availability = the percentage of time a system is **operational and able to serve requests correctly**, over a given period.

```
Availability = Uptime / (Uptime + Downtime)
```

It answers: _"If I hit this system right now, will I get a valid response?"_

---

## 2. The "Nines" Table (memorize this for interviews)

|Availability|Downtime/year|Downtime/month|Downtime/day|
|---|---|---|---|
|90% (one nine)|36.5 days|3 days|2.4 hrs|
|99% (two nines)|3.65 days|7.2 hrs|14.4 min|
|99.9% (three nines)|8.76 hrs|43.8 min|1.44 min|
|99.99% (four nines)|52.6 min|4.32 min|8.6 sec|
|99.999% (five nines)|5.26 min|25.9 sec|0.86 sec|

**Interview tip:** most production SaaS SLAs target 99.9–99.99%. Five nines is reserved for telecom/critical infra and is _extremely_ expensive to achieve.

---

## 3. How to Improve Availability

| Technique                              | How it helps                                                                              |
| -------------------------------------- | ----------------------------------------------------------------------------------------- |
| **Redundancy**                         | Multiple instances/replicas — no single point of failure (SPOF) (Single Point of Failure) |
| **Failover**                           | Automatic switch to a healthy replica/region on failure                                   |
| **Load balancing**                     | Distributes load, routes around unhealthy nodes                                           |
| **Health checks**                      | Detect failures fast so traffic can be rerouted                                           |
| **Multi-AZ / multi-region deployment** | Survive datacenter or regional outages                                                    |
| **Graceful degradation**               | Serve partial/cached functionality instead of full failure                                |
| **Circuit breakers**                   | Stop cascading failures by failing fast on a broken dependency                            |
| **Rolling deployments**                | Avoid downtime during releases                                                            |

---

## 4. Availability in Redundant Systems (Math)

For **independent** components:

- **Series** (both must work): `A_total = A1 × A2` → adding components in series _reduces_ overall availability.
- **Parallel** (either can work): `A_total = 1 - (1-A1)(1-A2)` → adding redundant parallel components _increases_ overall availability.

**Example:** two servers each at 99% availability, load-balanced (parallel): `1 - (0.01 × 0.01) = 1 - 0.0001 = 99.99%`

This is _why_ redundancy is the #1 lever for availability.

---

## 5. Availability vs Reliability vs Fault Tolerance

|Term|Focus|
|---|---|
|**Availability**|Is the system _up and responding_ right now?|
|**Reliability**|Does the system work _correctly, consistently over time_ without failure?|
|**Fault tolerance**|Can the system _keep working correctly_ even when parts fail?|

A system can be available but unreliable (up, but returning wrong/flaky results occasionally) — see [[reliability]].

---

## 6. SLA vs SLO vs SLI

|Term|Meaning|
|---|---|
|**SLI** (Indicator)|Actual measured metric, e.g. "99.95% of requests succeeded last 30 days"|
|**SLO** (Objective)|Internal target, e.g. "99.9% success rate"|
|**SLA** (Agreement)|External contract with penalties, e.g. "99.9% uptime or credits issued"|

## SLI vs SLO vs SLA

|                            | ## Service Level Indicator               | ## Service Level Objective                                   | ## Service Level Agreement                  |
| -------------------------- | ---------------------------------------- | ------------------------------------------------------------ | ------------------------------------------- |
| **Component**              | **SLI**                                  | **SLO**                                                      | **SLA**                                     |
| **Definition**             | A metric that shows service performance. | A target set for an SLI                                      | A contract with customers                   |
| **Purpose**                | Measures service performance             | Sets the performance goal for SLIs                           | Formal commitment to customers              |
| **Focus**                  | Focus on service health                  | Focus on the performance target                              | Focus on customer commitment                |
| **Examples**               | Latency, error rate, throughput          | 99.9% uptime, <200ms latency                                 | Uptime guarantees, support response time    |
| **Audience**               | Teams / Engineers                        | Teams / Engineers                                            | Customers                                   |
| **Measurement**            | Real-time                                | Monthly/Quarterly                                            | Monthly/Quarterly                           |
| **Legally Actionable**     | No, it’s not legally actionable          | No, it’s not legally actionable                              | Yes, it’s legally actionable                |
| **Flexibility**            | High – It can track many metrics.        | Medium – It can be adjusted for each service or time period. | Low – Because of legal binding              |
| **When to Use**            | Use to monitor the system continuously.  | Use to guide internal reliability goals.                     | Use for customer agreements and guarantees. |
| **Error Budget Relevance** | Provides data to set SLOs                | Defines allowed failure (error budget)                       | Penalties apply if breached                 |


SLO is usually set _tighter_ than SLA to leave error-budget buffer.

---

## 7. Error Budgets

`Error budget = 1 - SLO`

If SLO = 99.9%, error budget = 0.1% of requests/time allowed to fail. Teams use this to balance **feature velocity vs stability** — if the budget is unspent, ship faster; if exhausted, freeze releases and focus on reliability.

---

## 8. Interview Talking Points

- Availability and consistency are often in tension in distributed systems ([[cap-theorem]]).
- High availability usually costs more (redundant infra, cross-region replication, ops complexity) — always state this trade-off explicitly in interviews.
- "How would you achieve 99.99% availability for this system?" → talk about redundancy, health checks, multi-region failover, and load balancing together, not just one.

---

## 9. Resources

- [Google SRE Book – Ch. 4: Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [AWS Well-Architected – Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)
- [Uptime.is – downtime calculator by nines](https://uptime.is/)

---

## Related Notes

- [[reliability]]
- [[cap-theorem]]
- [[scalability]]
- [[system-design-networking-index]]