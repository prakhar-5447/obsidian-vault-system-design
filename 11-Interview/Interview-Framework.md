# System Design Interview Framework

## 1. Clarify Requirements

- Who are the users?
- What are the core features?
- What scale are we targeting?
- What consistency is required?
- What latency is acceptable?

## 2. Estimate Scale

Calculate:

- DAU
- Requests/sec
- Peak RPS
- Storage
- Bandwidth
- Read/write ratio

## 3. Define APIs

## 4. Define Data Model

## 5. Draw High-Level Architecture

```text
Client
 ↓
Load Balancer
 ↓
API Gateway
 ↓
Services
 ↓
Cache / Database / Queue
```

## 6. Identify Bottlenecks

- Database
- CPU
- Network
- Cache
- Queue

## 7. Scale

- Caching
- Replication
- Sharding
- Load balancing
- CDN
- Async processing

## 8. Reliability

- Retry
- Timeout
- Idempotency
- Circuit breaker
- Failover

## 9. Deep Dive

Pick the hardest part of the system and design it deeply.

## 10. Trade-offs

Always explain why you selected a particular approach.
