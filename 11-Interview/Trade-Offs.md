# Common System Design Trade-offs

| Choice | Advantage | Cost / Trade-off |
|---|---|---|
| SQL | Strong transactions, relational queries | Horizontal scaling can be harder |
| NoSQL | Flexible schema, horizontal scale | Consistency/query trade-offs |
| Cache | Low latency | Stale data / invalidation |
| Sync processing | Simple consistency | Higher request latency |
| Async processing | Better decoupling and throughput | More complexity |
| Strong consistency | Fresh reads | Higher coordination cost |
| Eventual consistency | Availability / scale | Stale reads |
| Monolith | Simple deployment | Harder independent scaling |
| Microservices | Independent scaling/deployment | Operational complexity |
| REST | Simple, broadly supported | More overhead for some internal calls |
| gRPC | Efficient service-to-service calls | More tooling/protocol complexity |
