# Networking — Overview

## 1. What This Note Covers

The mental model that ties together the networking notes in this vault: [[dns]], [[tcp]], [[udp]], [[http]], [[https]], [[rest]], [[grpc]], [[websocket]]. Start here, then drill into the specific note when you need depth.

---

## 2. The OSI Model (Simplified, Interview-Relevant Layers)

Real systems mostly care about layers 3-7. Know which layer each protocol lives at — a common quick interview check.

|Layer|Name|Concern|Examples|
|---|---|---|---|
|7|Application|What the data _means_|HTTP, HTTPS, gRPC, DNS, WebSocket|
|4|Transport|Getting bytes between processes reliably (or not)|TCP, UDP|
|3|Network|Getting packets between machines (routing)|IP|
|2|Data Link|Getting frames between adjacent devices|Ethernet, WiFi|
|1|Physical|Actual bits on the wire/air|Cables, radio|

**Interview shorthand:** "Please Do Not Throw Sausage Pizza Away" (Physical, Data Link, Network, Transport, Session, Presentation, Application) — most system design discussions only ever touch Network/Transport/Application (3, 4, 7).

---

## 3. How a Request Actually Travels (Putting It All Together)

A browser hitting `https://api.example.com/users/123`:

```
1. DNS resolution          → api.example.com → IP address        [[dns]]
2. TCP handshake            → 3-way handshake, ~1 RTT              [[tcp]]
3. TLS handshake             → encrypt the connection, ~1 RTT       [[https]]
4. HTTP request sent         → GET /users/123 over the connection   [[http]]
5. Server processes          → maybe calls internal gRPC services   [[grpc]]
6. HTTP response returned    → JSON body, status code                [[rest]]
7. Connection kept alive     → reused for next request (avoid re-handshake)
```

Every step above has its own latency cost — see [[latency]]'s breakdown table for actual numbers. This request path is the backbone reference to mentally replay whenever a system design question starts with "a user makes a request."

---

## 4. Protocol Decision Cheat Sheet

| Need                                                            | Reach for                                 |
| --------------------------------------------------------------- | ----------------------------------------- |
| Public API, broad client compatibility                          | [[rest]] over [[https]]                   |
| Internal service-to-service, high performance, strict contracts | [[grpc]]                                  |
| Bidirectional real-time (chat, live collaboration)              | [[websocket]]                             |
| Server-to-client only real-time feed                            | SSE (see [[websocket]] comparison table)  |
| Reliable ordered delivery                                       | [[tcp]] (underlies HTTP, gRPC, WebSocket) |
| Low-overhead, loss-tolerant real-time (video, gaming, VoIP)     | [[udp]]                                   |
| Name → IP resolution, geo-routing, traffic steering             | [[dns]]                                   |
| Encrypting any of the above                                     | [[https]] (TLS)                           |

---

## 5. Transport Layer Foundation`

Everything at layer 7 sits on top of a layer 4 choice:`

|                               | Built on TCP   | Built on UDP                                       |
| ----------------------------- | -------------- | -------------------------------------------------- |
| HTTP/1.1, HTTP/2              | ✅              |                                                    |
| HTTP/3 (QUIC)                 |                | ✅ (QUIC adds its own reliability atop UDP)         |
| gRPC                          | ✅ (via HTTP/2) |                                                    |
| WebSocket                     | ✅              |                                                    |
| DNS                           |                | ✅ (usually; falls back to TCP for large responses) |
| Video/voice streaming, gaming |                | ✅                                                  |

`The recurring theme across [[tcp]] and [[udp]]: **TCP's strict ordering guarantee causes head-of-line blocking**; protocols that can't tolerate that (real-time media) or want to avoid it at scale (HTTP/3/QUIC) build atop UDP instead.`

---

## 6. Where Networking Meets System Design Fundamentals

| Networking concept                           | Connects to                                               |
| -------------------------------------------- | --------------------------------------------------------- |
| DNS routing policies (GeoDNS, latency-based) | [[availability]], load balancing                          |
| TCP/TLS handshake cost                       | [[latency]]                                               |
| HTTP statelessness                           | [[scalability]] (trivial horizontal scaling)              |
| WebSocket's stateful connections             | [[scalability]] (needs sticky sessions + pub/sub fan-out) |
| gRPC streaming vs REST                       | [[throughput]] vs simplicity trade-off                    |
| CAP-theorem-style choices in DNS failover    | [[cap-theorem]], [[reliability]]                          |

---

## 7. Interview Framework: "Design the Communication Layer"

When a system design question touches networking, walk through:

1. **Public-facing or internal?** → REST/HTTPS vs gRPC
2. **Request-response or streaming/real-time?** → REST vs WebSocket/SSE vs gRPC streaming
3. **Latency-sensitive?** → consider UDP-based options, CDN/edge placement, connection reuse
4. **What's the failure/routing story?** → DNS-based failover, health checks, load balancing
5. **What's encrypted, and where does TLS terminate?** → usually at the LB/CDN edge

---

## 8. Resources

- _High Performance Browser Networking_ by Ilya Grigorik — free online, covers TCP/UDP/TLS/HTTP end-to-end: https://hpbn.co/
- [Cloudflare Learning Center](https://www.cloudflare.com/learning/) — excellent plain-language explainers for every protocol in this note
- [System Design Primer (GitHub)](https://github.com/donnemartin/system-design-primer)

---

## Related Notes

- [[dns]]
- [[tcp]]
- [[udp]]
- [[http]]
- [[https]]
- [[rest]]
- [[grpc]]
- [[websocket]]
- [[latency]]
- [[scalability]]