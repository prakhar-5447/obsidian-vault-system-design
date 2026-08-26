# TCP (Transmission Control Protocol)

## 1. Definition

TCP is a **connection-oriented, reliable, ordered** transport-layer protocol (OSI Layer 4). It guarantees that bytes sent arrive **in order, without duplication or loss**, or the connection reports an error.

Almost everything on the internet that needs reliability (HTTP, HTTPS, gRPC, most databases) runs over TCP.

---

## 2. Three-Way Handshake (Connection Setup)

```
Client                    Server
  |------ SYN ------------->|
  |<---- SYN-ACK ------------|
  |------ ACK ------------->|
  |    connection established
```

1. Client sends `SYN` (synchronize) with initial sequence number
2. Server responds `SYN-ACK` (acknowledges + sends its own sequence number)
3. Client responds `ACK` — connection is now open

**Cost:** 1 full round-trip (RTT) before any data is sent — this is a major latency contributor for short-lived connections (see [[latency]]).

Teardown uses a **four-way handshake** (`FIN`/`ACK` from each side) since either side can close independently.

---

## 3. How TCP Guarantees Reliability

|Mechanism|Purpose|
|---|---|
|**Sequence numbers**|Reorder out-of-order packets on arrival|
|**Acknowledgments (ACK)**|Confirm receipt; unacked data gets retransmitted|
|**Checksums**|Detect corrupted packets|
|**Retransmission (timeout-based)**|Resend lost packets|
|**Flow control** (sliding window)|Prevent sender from overwhelming a slow receiver|
|**Congestion control** (e.g. TCP Reno, CUBIC, BBR)|Prevent overwhelming the _network_, back off on packet loss|

---

## 4. Flow Control vs Congestion Control

| |Flow Control|Congestion Control|
|---|---|---|
|Protects|The **receiver** (don't overwhelm it)|The **network** (don't overwhelm links/routers)|
|Mechanism|Receiver advertises window size|Sender infers network state (loss/latency) and adjusts send rate|
|Algorithms|Sliding window|Slow start, congestion avoidance, CUBIC, BBR|

---

## 5. Head-of-Line (HOL) Blocking

Because TCP guarantees **strict ordering**, if one packet is lost, **all subsequent packets wait** for it to be retransmitted — even if they already arrived. This is a fundamental TCP limitation that motivated HTTP/3's move to QUIC (UDP-based) — see [[udp]] and [[http]].

---

## 6. TCP vs UDP — Quick Comparison

| |TCP|UDP|
|---|---|---|
|Connection|Connection-oriented (handshake)|Connectionless|
|Reliability|Guaranteed delivery, ordering|No guarantees|
|Speed|Slower (overhead of guarantees)|Faster (minimal overhead)|
|Use case|Web, email, file transfer|Video streaming, gaming, DNS, VoIP|

Full breakdown in [[udp]].

---

## 7. When TCP Is the Wrong Choice

- Real-time media (video calls, live streaming) — a late/retransmitted packet is _worse_ than a dropped frame; prefer UDP-based protocols.
- High-frequency small messages where HOL blocking hurts more than occasional loss — motivated QUIC/HTTP3.

---

## 8. Interview Talking Points

- Explain the handshake cost when discussing connection reuse (keep-alive, connection pooling) as a latency optimization.
- Mention HOL blocking when comparing HTTP/2 (still TCP, still has HOL blocking at the TCP layer despite multiplexing streams) vs HTTP/3 (QUIC/UDP, solves this).
- Congestion control matters when discussing **bulk data transfer performance** across high-latency links (e.g. cross-region replication).

---

## 9. Resources

- [Cloudflare Learning – What is TCP?](https://www.cloudflare.com/learning/ddos/glossary/tcp-ip/)
- [High Performance Browser Networking – Ilya Grigorik (free online book), Ch. 2](https://hpbn.co/building-blocks-of-tcp/)
- [RFC 9293 – TCP (current spec)](https://www.rfc-editor.org/rfc/rfc9293)

---

## Related Notes

- [[udp]]
- [[http]]
- [[https]]
- [[websocket]]
- [[latency]]
- [[system-design-networking-index]]