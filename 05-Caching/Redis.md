# Redis

## 1. Definition

Redis (**RE**mote **DI**ctionary **S**erver) is an **in-memory data structure store**, used as a cache, database, message broker, and streaming engine. Single-threaded core (per shard) with an event loop — extremely fast because data lives in RAM and the design avoids lock contention.

The most common real-world implementation of the caching concepts in [[caching]], [[cache-strategies]], [[cache-invalidation]], and [[cache-stampede]].

---

## 2. Core Data Structures (Beyond Simple Key-Value)

|Type|Use case|Example commands|
|---|---|---|
|**String**|Simple cache values, counters|`SET`, `GET`, `INCR`|
|**Hash**|Object-like data (e.g. a user record) without a full JSON blob|`HSET`, `HGET`, `HGETALL`|
|**List**|Queues, recent-activity feeds|`LPUSH`, `RPOP`, `LRANGE`|
|**Set**|Unique collections, tag membership, intersection/union ops|`SADD`, `SINTER`, `SUNION`|
|**Sorted Set (ZSet)**|Leaderboards, rate limiting, priority queues, ranked feeds|`ZADD`, `ZRANGE`, `ZRANGEBYSCORE`|
|**Stream**|Append-only log, event sourcing, lightweight Kafka alternative|`XADD`, `XREAD`, `XGROUP`|
|**HyperLogLog**|Approximate unique-count (cardinality estimation) at tiny memory cost|`PFADD`, `PFCOUNT`|
|**Bitmap**|Space-efficient boolean flags (e.g. daily active user tracking)|`SETBIT`, `BITCOUNT`|
|**Geospatial**|Location-based queries|`GEOADD`, `GEOSEARCH`|

**Interview-worthy example — leaderboard with ZSet:**

```
ZADD leaderboard 1500 "player1"
ZADD leaderboard 2200 "player2"
ZREVRANGE leaderboard 0 9 WITHSCORES   # top 10 players
```

O(log N) insert, O(log N + M) range fetch — far better than sorting in application code on every read.

---

## 3. Persistence Options

Redis is in-memory by default (data lost on restart), but supports optional persistence:

|Method|How it works|Trade-off|
|---|---|---|
|**RDB (snapshotting)**|Periodic point-in-time snapshot written to disk|Fast restarts, but can lose data since the last snapshot|
|**AOF (Append-Only File)**|Logs every write operation, replayed on restart|More durable (configurable fsync: every write / every second), but larger files, slower restarts|
|**RDB + AOF combined**|Common production setup|Best durability/performance balance|
|**None**|Pure cache, no persistence|Fastest, but any restart = full cold cache (see [[cache-stampede]] risk on cold start)|

**Interview point:** if Redis is used purely as a cache (source of truth is elsewhere, e.g. Postgres), persistence often isn't critical — data loss just means more cache misses temporarily. If Redis is the **primary data store** for something, persistence + replication become essential.

---

## 4. Redis for Distributed Locking

```
SET lock:resource_1 "unique_token" NX EX 10
```

`NX` (only set if not exists) + `EX` (auto-expiring TTL) implements a simple distributed lock — used directly in the [[cache-stampede]] mutex pattern.

**Redlock algorithm**: for locks needing stronger guarantees across a Redis cluster (not just a single node), acquiring the lock on a **majority of independent Redis nodes** — though this algorithm has known debated edge cases (see Martin Kleppmann's critique in resources) and isn't bulletproof for correctness-critical locking (e.g. don't rely on it alone for financial transactions).

---

## 5. Scaling Redis

|Approach|How it works|
|---|---|
|**Replication**|Primary-replica setup (like [[replication]]) — replicas serve reads, primary handles writes, automatic failover via Redis Sentinel|
|**Redis Cluster**|Native sharding — data split across multiple nodes using **hash slots** (16,384 fixed slots, `CRC16(key) % 16384` decides slot); each node owns a range of slots|
|**Redis Sentinel**|Monitors primary/replica health, handles automatic failover — doesn't shard, just adds HA to a replica set|

**Redis Cluster hash slots** are conceptually similar to (but simpler than) the consistent hashing described in [[sharding]] — fixed number of slots makes rebalancing (moving slot ranges between nodes) more predictable than a hash ring.

---

## 6. Eviction Policies (When Memory Is Full)

Configurable via `maxmemory-policy`:

|Policy|Behavior|
|---|---|
|`noeviction`|Reject new writes once full (errors instead of evicting)|
|`allkeys-lru`|Evict least-recently-used key across all keys|
|`volatile-lru`|Evict LRU among keys **with a TTL set** only|
|`allkeys-lfu`|Evict least-frequently-used key|
|`volatile-ttl`|Evict the key with the **shortest remaining TTL** first|
|`allkeys-random` / `volatile-random`|Evict a random key|

Maps directly onto the general eviction policy concepts in [[caching]] — Redis just makes them configurable per deployment.

---

## 7. Common Redis Use Cases in System Design

| Use case                            | Redis feature used                                                                                       |
| ----------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Application cache (cache-aside)     | Strings/Hashes + TTL                                                                                     |
| Session storage                     | Strings/Hashes + TTL                                                                                     |
| Rate limiting                       | `INCR` + `EXPIRE`, or sorted sets for sliding window                                                     |
| Leaderboards                        | Sorted sets                                                                                              |
| Pub/Sub messaging                   | `PUBLISH`/`SUBSCRIBE` (lightweight, no persistence — not a replacement for [[message-queues]]            |
| Distributed locks                   | `SET NX EX`                                                                                              |
| Real-time analytics (unique counts) | HyperLogLog                                                                                              |
| Job/task queues                     | Lists (`LPUSH`/`BRPOP`) — simple queue, though dedicated queue systems are more robust for complex needs |

---

## 8. Redis vs Memcached (Common Interview Comparison)

| |Redis|Memcached|
|---|---|---|
|Data structures|Rich (strings, hashes, sets, sorted sets, streams)|Simple key-value only|
|Persistence|Optional (RDB/AOF)|None (pure in-memory, always)|
|Replication/clustering|Built-in|Not built-in (client-side sharding only)|
|Multi-threading|Mostly single-threaded (I/O threading added in newer versions for networking)|Multi-threaded natively|
|Pub/Sub, locks, streams|Yes|No|
|When to choose|Need more than simple caching (structures, persistence, pub/sub)|Pure, maximally simple caching layer, nothing else|

---

## 9. Interview Talking Points

- Bring up the **specific data structure** that fits the problem (sorted sets for leaderboards/rate-limiting, HyperLogLog for approximate unique counts) — shows depth beyond "Redis = cache."
- Mention **persistence trade-offs** only when relevant — don't over-engineer durability for a pure cache use case.
- When asked about Redis at scale, mention **Redis Cluster's hash slots** specifically, and connect it to the general sharding concepts in [[sharding]].
- Distinguish Redis Pub/Sub (fire-and-forget, no persistence, message lost if no subscriber is listening) from real message queues like Kafka/RabbitMQ when durability/replay matters — see [[message-queues]].

---

## 10. Resources

- [Redis Official Docs](https://redis.io/docs/latest/)
- [Redis University (free courses)](https://university.redis.com/)
- [Martin Kleppmann – "How to do distributed locking" (Redlock critique)](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [AWS ElastiCache Docs (managed Redis)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/WhatIs.html)

---

## Related Notes

- [[caching]]
- [[cache-strategies]]
- [[cache-invalidation]]
- [[cache-stampede]]
- [[sharding]]
- [[replication]]
- [[message-queues]]