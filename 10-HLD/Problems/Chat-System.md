# Chat-System

## 1. Requirements

### Functional
* 1-on-1 direct messaging and group chats (up to 500 members).
* Real-time online/offline presence and typing indicators.
* Message delivery receipts (Sent, Delivered, Read).
* Media attachment support (images, videos, documents via pre-signed URLs).
* Push notifications for offline recipients (APNs/FCM).
* Message history retrieval with cursor-based pagination.

### Non-Functional
* **Availability:** 99.999% uptime; prioritize partition tolerance and availability over cross-region strong consistency.
* **Latency:** Sub-100 ms for real-time delivery over active WebSocket sessions.
* **Consistency:** Strict chronological ordering within a conversation; eventual consistency for presence and read receipts.
* **Scalability:** Support 50M+ Daily Active Users (DAU) and millions of concurrent WebSocket connections.

---

## 2. Capacity Estimation

* **DAU:** 50 Million
* **Requests/sec:** ~12,000 msgs/sec average (assuming 20 messages/user/day $\\approx$ 1 Billion msgs/day)
* **Peak RPS:** ~120,000 msgs/sec (10× peak traffic multiplier)
* **Storage:** ~1 TB/day raw message data (1 KB/msg); ~1 PB/year effective storage including indexes, replication, and metadata
* **Bandwidth:** Ingress $\\approx$ 11.6 MB/s; Egress $\\approx$ 58–116 MB/s (due to 1-to-many fan-out and multi-device sync)

---

## 3. APIs

```http
# Real-Time WebSocket Endpoint
WSS /ws/v1/chat

# Send a Message (REST / WS payload)
POST /api/v1/conversations/{conversationId}/messages
Payload: {
  "clientMessageId": "uuid-v4",
  "type": "TEXT | IMAGE | FILE",
  "content": "Hello world",
  "mediaUrl": "[https://cdn.chat.com/media/xyz](https://cdn.chat.com/media/xyz)"
}

# Fetch Historical Messages (Cursor-based)
GET /api/v1/conversations/{conversationId}/messages?cursor={messageId}&limit=50

# Presence Heartbeat
POST /api/v1/presence/heartbeat
Payload: { "status": "ONLINE | IDLE" }
## 4. Data Model

```text
Define entities / tables here.
```
## 4. Data Model

```sql
-- Conversations Table
CREATE TABLE conversations (
    conversation_id UUID PRIMARY KEY,
    type TEXT, -- 'DIRECT' | 'GROUP'
    name TEXT,
    created_at TIMESTAMP
);

-- Conversation Members
CREATE TABLE conversation_members (
    conversation_id UUID,
    user_id UUID,
    role TEXT, -- 'ADMIN' | 'MEMBER'
    last_delivered_seq BIGINT,
    last_read_seq BIGINT,
    PRIMARY KEY (conversation_id, user_id)
);

-- Messages Table (Partitioned by conversation_id, sorted by sequence/time)
CREATE TABLE messages (
    conversation_id UUID,
    message_id TIMEUUID, -- Snowflake ID
    sender_id UUID,
    client_message_id UUID,
    content TEXT,
    media_url TEXT,
    sequence_number BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (conversation_id, sequence_number)
) WITH CLUSTERING ORDER BY (sequence_number DESC);
```


## 5. High-Level Architecture

```text
Client (Web / Mobile)
       │
       ▼
[ DNS / CDN (Cloudflare) ]
       │
       ▼
[ TCP / TLS Load Balancers (HAProxy / Envoy) ]
       │
 ┌─────┴───────────────────────────┐
 │ (HTTP / REST)                   │ (Persistent WebSockets)
 ▼                                 ▼
[ API Gateway / Auth ]       [ Chat Gateway Servers ]
 │                                 │
 ├── User Service                  ├── Presence Service (Redis Cluster)
 ├── Channel Service               ├── Kafka (Message Ingestion Buffer)
 └── Media Service (S3)            │
                                   ▼
                             [ Delivery Workers ]
                                   │
                    ┌──────────────┴──────────────┐
                    ▼                             ▼
        [ Cassandra / ScyllaDB ]         [ Push Notifications ]
         (Persistent History)            (APNs / FCM via SQS)
```

## 6. Deep Dive

### Component 1: Stateful WebSocket Gateway & Session Registry

- **Connection Tier:** Clients maintain persistent full-duplex WebSocket connections to distributed gateway nodes.
    
- **Session Registry:** A distributed Redis cache maps `userId -> { serverId, connectionId }`.
    
- **Routing Path:** Ingesting nodes publish messages to Kafka (partitioned by `conversationId`). Delivery workers consume messages, lookup recipient nodes via the Redis Session Registry, and dispatch frames via internal gRPC/Redis PubSub. Offline users trigger push notification jobs.
    

### Component 2: Monotonic Sequence Numbering & Idempotency

- **Deterministic Ordering:** Each conversation increments a per-channel sequence number (or generates a 64-bit Snowflake ID). This ensures deterministic ordering without requiring global cross-cluster clock synchronization.
    
- **Write Deduplication:** The pair `(conversationId, clientMessageId)` has a unique index/deduplication key in cache to drop duplicate packets caused by network retries.

## 7. Scaling

- **WebSocket Gateway Tier:** Horizontally scaled using non-blocking I/O event loops (epoll/kqueue) capable of supporting 100K+ concurrent idle connections per host.
    
- **Database Sharding:** Messages partitioned by `conversationId`. For ultra-active channels, monthly bucket compounds `(conversationId, year_month)` prevent partitions from exceeding 100 MB.
    
- **Group Fan-out Strategy:**
    
    - **Small Groups (<100 members):** Fan-out on Write (immediate delivery event pushed to all member queues).
        
    - **Large Groups (>1,000 members):** Fan-out on Read (message stored once; clients pull/sync delta on active view).

## 8. Failure Handling

- **Retry:** Exponential backoff with jitter on client connection retries and dropped message handshakes.
    
- **Timeout:** 30-second WebSocket ping/pong heartbeats to detect dead/half-open TCP sockets and purge stale Redis sessions.
    
- **Idempotency:** Client-generated `clientMessageId` enforces at-most-once processing at the gateway.
    
- **Circuit Breaker:** Applied to third-party push gateways (FCM/APNs) to prevent worker pool starvation during provider downtime.
    
- **Failover:** Load balancer automatically shifts dropped connections to surviving gateways; state is rebuilt on handshake from the central session store.
    

## 9. Bottlenecks

- **Redis Presence Stampedes:** Broadcast-on-change presence creates $O(N^2)$ traffic spikes in large networks. _Mitigation:_ Pull-based presence on chat-view open and heartbeat TTL expiry.
    
- **High Write IOPS:** Random disk writes degrade traditional relational databases. _Mitigation:_ LSM-tree storage engines (Cassandra/ScyllaDB) that write sequentially to commit logs and memtables.
    

## 10. Trade-offs

- **Cassandra vs Relational DB:** Cassandra provides horizontal scale and superior LSM write append throughput at the expense of multi-table ACID transactions.
    
- **WebSockets vs HTTP Long Polling:** WebSockets require stateful infrastructure and connection management but minimize header overhead and latency.
    
- **Synchronous Ack + Async Fan-Out:** Message persistence is acknowledged synchronously to the sender, while multi-device and offline delivery are handled asynchronously via Kafka.
    

## 11. Interview Walkthrough

1. **Clarify Scope & Scale (1–2 min):** Outline functional goals (1-to-1, groups, receipts, presence) and establish traffic constraints (50M DAU, 1B msgs/day, 120K peak RPS, sub-100 ms delivery).
    
2. **Architecture & Protocol (2–3 min):** Contrast stateless REST APIs (auth, media upload, history) with stateful WebSocket gateways. Walk through the Redis session registry and Kafka ingestion pipeline.
    
3. **Data Layer & Ordering (2 min):** Explain why Cassandra fits write-heavy append-only chat data with partition key `conversationId` and clustering key `sequence_number`.
    
4. **Resilience & Edge Cases (2 min):** Cover duplicate mitigation via `clientMessageId`, heartbeat-based dead connection pruning, and hybrid fan-out strategies for large groups.
    

Explain the system in 5–10 minutes without notes.
