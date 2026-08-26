# HTTPS (HTTP Secure)

## 1. Definition

HTTPS = [[http]] layered on top of **TLS (Transport Layer Security)**, providing:

- **Encryption** — data can't be read in transit (protects against eavesdropping)
- **Integrity** — data can't be tampered with in transit (detects modification)
- **Authentication** — proves you're talking to the real server, not an impersonator (via certificates)

`HTTP + TLS = HTTPS`. Same HTTP semantics, encrypted transport underneath.

---

## 2. TLS Handshake (Simplified — TLS 1.3)

```
Client                              Server
  |----- ClientHello -------------->|  (supported ciphers, random value)
  |<---- ServerHello + Cert --------|  (chosen cipher, cert, key share)
  |----- Finished ------------------>|
  |    encrypted application data flows
```

- **TLS 1.2**: required 2 round trips before data could flow
- **TLS 1.3**: reduced to **1 round trip** (or **0-RTT** for resumed connections) — a meaningful latency win at scale, see [[latency]]

Combined with the TCP handshake, a fresh HTTPS connection historically cost **~2-3 RTTs** before the first byte of data — this is why **connection reuse (keep-alive)** and **TLS session resumption** matter so much in system design.

---

## 3. How the Certificate/Trust Chain Works

1. Server presents a **certificate** signed by a **Certificate Authority (CA)** (e.g. Let's Encrypt, DigiCert)
2. Browser/OS has a built-in list of **trusted root CAs**
3. Browser verifies the chain: `Root CA → Intermediate CA → Server Cert` — if it traces back to a trusted root, the cert is valid
4. Certificate includes the server's **public key**, used to establish a shared symmetric key for the session (asymmetric crypto for handshake, symmetric for bulk data — symmetric is much faster)

**Common cert issues to know:** expired certs, hostname mismatch, self-signed certs (not trusted by browsers), and mixed content (loading HTTP resources on an HTTPS page — browsers block/warn).

---

## 4. Why HTTPS Everywhere (Not Just Login Pages)

- Prevents **man-in-the-middle (MITM)** attacks, packet sniffing on public WiFi
- Prevents ISPs/networks from injecting ads or tracking scripts into pages
- Required for modern browser features (HTTP/2, geolocation, service workers)
- SEO ranking factor (Google favors HTTPS)
- Browsers now flag plain HTTP as "Not Secure"

---

## 5. Performance Optimizations for HTTPS

| Technique                                                  | Benefit                                                                                                                 |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **TLS session resumption** (session tickets/IDs)           | Skip full handshake on reconnect                                                                                        |
| **TLS 1.3 0-RTT**                                          | Send data with the very first packet on resumed connections (trade-off: replay-attack risk for non-idempotent requests) |
| **OCSP stapling** (**Online Certificate Status Protocol**) | Server includes cert revocation status, avoiding extra client round trip                                                |
| **HTTP/2 or HTTP/3**                                       | Amortize handshake cost across multiplexed requests on one connection                                                   |
| **CDN/edge termination**                                   | TLS terminated close to the user, reducing handshake RTT                                                                |

---

## 6. mTLS (Mutual TLS) — System Design Context

Standard HTTPS only authenticates the **server** to the client. **mTLS** requires the **client** to also present a certificate — common in:

- Service-to-service auth inside microservice meshes (e.g. Istio, Linkerd)
- Zero-trust internal networks (no implicit trust even inside the VPC)
- B2B APIs requiring strong client identity verification

---

## 7. Interview Talking Points

- Know the handshake cost trade-off and how TLS session resumption / TLS 1.3 mitigate it — relevant whenever latency-sensitive systems are discussed.
- Mention **TLS termination point** in architecture diagrams — often terminated at the load balancer/CDN edge, with internal traffic sometimes plaintext (or mTLS in zero-trust setups) — a good system-design detail to volunteer.
- Certificate rotation/expiry is a real operational failure mode — worth mentioning when discussing reliability/on-call scenarios.

---

## 8. Resources

- [Cloudflare Learning – What happens in a TLS handshake?](https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/)
- [High Performance Browser Networking – Ch. 4 (TLS)](https://hpbn.co/transport-layer-security-tls/)
- [Let's Encrypt – How It Works](https://letsencrypt.org/how-it-works/)
- [RFC 8446 – TLS 1.3 spec](https://www.rfc-editor.org/rfc/rfc8446)

---

## Related Notes

- [[http]]
- [[tcp]]
- [[dns]]
- [[latency]]
- [[system-design-networking-index]]