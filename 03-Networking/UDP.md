# UDP (User Datagram Protocol)

## 1. Definition

UDP is a **connectionless, unreliable, unordered** transport-layer protocol (OSI Layer 4). It sends independent packets ("datagrams") with minimal overhead — **no handshake, no guaranteed delivery, no ordering, no congestion control** by default.

Trade-off: you give up reliability for **speed and low overhead**.

---

## 2. UDP vs TCP — Full Comparison

| |UDP|TCP|
|---|---|---|
|Connection setup|None (no handshake)|3-way handshake (~1 RTT)|
|Reliability|No guarantee — packets can be lost, duplicated, arrive out of order|Guaranteed, ordered delivery|
|Header overhead|8 bytes|20+ bytes|
|Speed|Faster (no handshake, no ACKs, no retransmission)|Slower (overhead of guarantees)|
|Flow/congestion control|None built-in (app must implement if needed)|Built-in|
|Head-of-line blocking|None (packets are independent)|Yes (strict ordering blocks later packets)|
|Use case|Streaming, gaming, VoIP, DNS, QUIC/HTTP3|Web (HTTP/1.1, HTTP/2), email, file transfer|

---

## 3. Why Use an "Unreliable" Protocol?

For many real-time use cases, a **late** packet is _worse than a lost one_ — by the time a retransmitted video frame arrives, it's already irrelevant. UDP lets applications:

- Drop stale data instead of waiting for retransmission
- Avoid TCP's head-of-line blocking entirely
- Build **custom** reliability/ordering only where actually needed (rather than paying for guarantees you don't need everywhere)

---

## 4. Common UDP Use Cases

|Use case|Why UDP|
|---|---|
|**DNS**|Small single-packet request/response; retry at app layer if no response|
|**VoIP / video calls**|Real-time — dropped/late packets skipped, not retransmitted|
|**Live video streaming**|Same — smooth playback > perfect delivery|
|**Online gaming**|Position updates are only useful if timely; stale ones are discarded|
|**QUIC (HTTP/3)**|Built on UDP to get custom reliability + avoid TCP HOL blocking — see [[http]]|
|**DHCP, SNMP, NTP**|Simple request/response, low overhead preferred|

---

## 5. Building Reliability on Top of UDP

Applications that need _some_ reliability but want to avoid TCP's full overhead/HOL blocking build custom logic:

- **QUIC** (used by HTTP/3): adds encryption, per-stream reliability, and congestion control — but multiplexed streams don't block each other like TCP does
- **RTP/RTCP** (media streaming): sequence numbers for detecting loss/jitter, but no retransmission of old frames
- **Custom game networking**: often uses "reliable UDP" implementations (e.g. ENet) — only retransmits _critical_ packets (e.g. "player died"), not high-frequency position updates

---

## 6. Interview Talking Points

- Don't just say "UDP is faster" — explain _why_ it's faster (no handshake, no ACK/retransmission bookkeeping, no ordering buffer) and what you give up.
- When asked to design a live chat/streaming/gaming system, this is the moment to bring up UDP + why TCP's guarantees would hurt real-time UX.
- Mention QUIC as the modern "best of both worlds" bridge — UDP transport, but with reliability and multiplexing built at the app layer, avoiding TCP's baked-in ordering constraint.

---

## 7. Resources

- [Cloudflare Learning – What is UDP?](https://www.cloudflare.com/learning/ddos/glossary/user-datagram-protocol-udp/)
- [RFC 768 – UDP spec (very short, worth skimming)](https://www.rfc-editor.org/rfc/rfc768)
- [High Performance Browser Networking – Ch. 3 (UDP)](https://hpbn.co/building-blocks-of-udp/)
- [Cloudflare – QUIC and HTTP/3 explainer](https://www.cloudflare.com/learning/performance/what-is-http3/)

---

## Related Notes

- [[tcp]]
- [[http]]
- [[websocket]]
- [[dns]]
- [[system-design-networking-index]]