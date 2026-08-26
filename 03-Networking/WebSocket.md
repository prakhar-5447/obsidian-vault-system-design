# WebSocket

## 1. Definition

WebSocket is a protocol providing a **full-duplex, persistent connection** between client and server over a **single TCP connection** — after an initial HTTP handshake, the connection is "upgraded" and both sides can send messages **at any time**, independently, without the request-response cycle HTTP requires.

Solves the problem: HTTP is fundamentally request-response (client always initiates) — bad fit for servers needing to **push** data to clients in real time.

---

## 2. The Handshake (HTTP → WebSocket Upgrade)

```
Client                                Server
  |-- GET /chat HTTP/1.1 ------------>|
  |   Upgrade: websocket               |
  |   Connection: Upgrade              |
  |   Sec-WebSocket-Key: <random>      |
  |<-- 101 Switching Protocols --------|
  |   Sec-WebSocket-Accept: <hash>     |
  |                                     |
  |<==== persistent full-duplex ====>|
  |      connection, both sides push  |
```

1. Client sends a normal HTTP GET request with `Upgrade: websocket` header
2. Server responds `101 Switching Protocols` if it supports WS
3. Connection is now a raw TCP socket speaking the WebSocket framing protocol — **not HTTP anymore**

Uses `ws://` (unencrypted) or `wss://` (WebSocket over TLS, encrypted — analogous to HTTP vs HTTPS).

---

## 3. WebSocket vs HTTP Polling vs Server-Sent Events (SSE)

| |Short Polling|Long Polling|SSE|WebSocket|
|---|---|---|---|---|
|Direction|Client → Server only|Client → Server only|Server → Client only|Both directions|
|Connection|New request every interval|Held open until data/timeout|Single long-lived HTTP connection|Single persistent TCP connection|
|Latency|High (poll interval delay)|Lower|Low|Lowest|
|Overhead|High (repeated HTTP headers/handshakes)|Medium|Low|Lowest (minimal framing overhead)|
|Browser support|Universal|Universal|Good (not IE)|Universal (modern)|
|Best for|Simple, infrequent updates|Fallback for older infra|One-way feeds (notifications, live scores)|Chat, gaming, collaborative editing|

**Interview tip:** don't default to WebSocket for everything — if data only flows server→client (e.g. live stock ticker, notifications), **SSE is simpler** (plain HTTP, auto-reconnect built into the browser API, works through more proxies/firewalls). Reach for WebSocket specifically when you need **bidirectional, low-latency** communication.

---

## 4. Use Cases

|Use case|Why WebSocket fits|
|---|---|
|**Chat applications**|Both sides send messages unpredictably, low latency needed|
|**Live collaborative editing** (Google Docs-style)|Frequent bidirectional small updates|
|**Multiplayer gaming**|Real-time state sync both directions (though often UDP-based for even lower latency — see [[udp]])|
|**Live dashboards/trading platforms**|Server pushes frequent updates; client may send subscribe/unsubscribe commands|
|**Notifications**|Often SSE is sufficient instead (one-directional)|

---

## 5. System Design Challenges with WebSocket

|Challenge|Why it matters|
|---|---|
|**Stateful connections**|Unlike stateless HTTP, each server holds an open connection per client — breaks naive horizontal scaling/load balancing|
|**Load balancer support**|LB must support persistent connections + sticky sessions (route the client back to the same server)|
|**Scaling out**|Need a pub/sub layer (Redis Pub/Sub, Kafka) so a message from any server can reach a client connected to a _different_ server|
|**Connection limits**|Each open WS connection consumes server memory/file descriptors — capacity planning differs from stateless HTTP|
|**Reconnection handling**|Client must handle disconnects/reconnects gracefully (network drops, server restarts) — no built-in retry like SSE|
|**Firewall/proxy issues**|Some corporate proxies/older infra don't handle the `Upgrade` header well|

**Architecture pattern for scaling WebSocket:** Load Balancer (sticky sessions) → WebSocket Gateway servers (hold connections) → Pub/Sub backend (Redis/Kafka) → any backend service publishes a message → gateway holding that client's connection receives it via subscription → pushes to client.

---

## 6. Interview Talking Points

- When designing a chat/real-time system, explicitly address: **how does a message from User A on Server 1 reach User B connected to Server 2?** → pub/sub fan-out is the expected answer.
- Mention the **stateful vs stateless** trade-off vs REST/HTTP — ties directly to [[scalability]] (stateless services scale trivially; stateful WS connections need sticky routing + connection-aware infra).
- Compare against **long polling** as a fallback for constrained environments (older clients, restrictive proxies).

---

## 7. Resources

- [MDN – WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)
- [RFC 6455 – The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455)
- [Ably – WebSockets vs SSE vs Long Polling](https://ably.com/topic/websockets-vs-sse)
- [Discord Engineering – Scaling WebSocket connections](https://discord.com/blog/how-discord-stores-trillions-of-messages) (broader real-time infra context)

---

## Related Notes

- [[http]]
- [[tcp]]
- [[udp]]
- [[grpc]]
- [[scalability]]
- [[system-design-networking-index]]