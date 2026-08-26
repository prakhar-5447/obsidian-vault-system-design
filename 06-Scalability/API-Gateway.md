# API Gateway

## 1. Definition

An API Gateway is a **single entry point** that sits in front of backend services, handling cross-cutting concerns (auth, rate limiting, routing, logging) before requests reach the actual services. Common in microservices architectures, where clients shouldn't need to know about — or directly call — dozens of individual internal services.

```
Client → API Gateway → [Auth check, rate limit, routing] → Service A / Service B / Service C
```

---

## 2. What an API Gateway Actually Does

|Responsibility|Why centralize it here|
|---|---|
|**Routing**|Maps `/users/*` → User Service, `/orders/*` → Order Service — client only knows one host|
|**Authentication / Authorization**|Validate tokens (JWT, OAuth) once, at the edge — services don't each reimplement this|
|**Rate limiting**|Enforce per-client/per-API limits centrally (see [[rate-limiting]])|
|**Request/response transformation**|Adapt payloads between what clients expect and what services return (e.g. legacy API version support)|
|**Aggregation / composition**|Combine responses from multiple services into one client-facing response (avoids client making many round trips)|
|**TLS termination**|Decrypt HTTPS at the edge, internal traffic can be plaintext or mTLS (see [[https]])|
|**Logging / monitoring / tracing**|Centralized observability entry point — one place to see all incoming traffic|
|**Caching**|Cache responses for cacheable endpoints at the gateway layer (see [[caching]])|
|**Load balancing**|Distribute requests across service instances (see [[load-balancing]])|
|**Circuit breaking**|Stop routing to a failing service, fail fast instead of cascading (see [[reliability]])|

---

## 3. Why Not Let Clients Call Services Directly?

Without a gateway:

- Client needs to know **every** service's address, and every change to internal architecture breaks clients
- Auth/rate-limiting logic duplicated across every service (or worse, inconsistently implemented)
- No single place to enforce consistent policies, observability, or handle cross-service response aggregation
- Exposes internal architecture directly to the public internet — larger attack surface

The gateway **decouples the client-facing API contract from internal service architecture** — services can be split, merged, or rewritten behind the gateway without clients noticing.

---

## 4. API Gateway vs Load Balancer vs Reverse Proxy

| |Load Balancer|Reverse Proxy|API Gateway|
|---|---|---|---|
|Primary job|Distribute traffic across replicas of **one** service|Forward requests, sometimes with caching/SSL termination|Route + apply policy across **many different** services|
|Awareness of API semantics|None (or basic, at L7)|Some (path-based routing)|High (auth, rate limits, request shaping, per-route logic)|
|Typical scope|Within one service's replica set|General purpose|Whole system's public API surface|

**In practice:** an API Gateway is a specialized reverse proxy with L7 awareness of API-level concerns; a load balancer is often used **behind** the gateway, in front of each individual service's replicas. These aren't mutually exclusive — they're usually layered together.

---

## 5. Backend for Frontend (BFF) Pattern

A variation where **each client type** (web, mobile, third-party partners) gets its **own** gateway/API layer, tailored to that client's specific needs (different payload shapes, different aggregation logic):

```
Web client   → Web BFF   → services
Mobile client → Mobile BFF → services
```

**Why:** mobile clients often want smaller, more aggregated payloads (fewer round trips over cellular networks) than a web client editing a rich dashboard — a single generic gateway API often can't serve both well.

---

## 6. Service Mesh vs API Gateway (Common Point of Confusion)

| |API Gateway|Service Mesh (e.g. Istio, Linkerd)|
|---|---|---|
|Traffic direction|**North-south** (external client → internal services)|**East-west** (service → service, internal)|
|Concerns|Client-facing auth, rate limiting, routing, aggregation|Internal service discovery, mTLS between services, retries, internal load balancing|
|Typical placement|Edge of the system|Sidecar proxies alongside every internal service|

**They solve different problems and are often used together**: gateway handles the public entry point, mesh handles internal service-to-service traffic policy.

---

## 7. Gateway as a Bottleneck / Single Point of Failure

Since **all** traffic flows through it, the gateway itself must be:

- Highly available (multiple gateway instances behind a load balancer — see [[load-balancing]])
- Fast — added latency at the gateway is added latency for every request (see [[latency]])
- Horizontally scalable (stateless gateway instances, see [[scalability]])

**Interview point:** always mention that the gateway needs its own redundancy story — a common oversight is designing a beautiful microservices architecture and forgetting the single gateway in front of it is now the new SPOF.

---

## 8. Real-World Examples

- **AWS API Gateway** — managed, integrates with Lambda, IAM auth, throttling built-in
- **Kong** — open-source, plugin-based (auth, rate limiting, logging as plugins)
- **Nginx / Envoy** — can be configured as a lightweight gateway
- **Netflix Zuul** — early influential open-source API gateway from Netflix's microservices era

---

## 9. Interview Talking Points

- Bring up an API Gateway whenever a design has **multiple backend services and external clients** — it's an expected component in most microservices system design answers.
- Distinguish it clearly from a **service mesh** if the interviewer probes on internal service-to-service traffic — shows precise understanding rather than buzzword-dropping.
- Mention the **BFF pattern** when a design has meaningfully different client types (mobile vs web vs partner APIs).
- Don't forget the gateway's own **availability/scaling** story — always call this out explicitly.

---

## 10. Resources

- [AWS – What is an API Gateway?](https://aws.amazon.com/api-gateway/)
- [Kong Docs](https://docs.konghq.com/)
- [Sam Newman – Backends For Frontends pattern](https://samnewman.io/patterns/architectural/bff/)
- [Istio Docs – Service Mesh concepts](https://istio.io/latest/docs/concepts/what-is-istio/)

---

## Related Notes

- [[load-balancing]]
- [[rate-limiting]]
- [[https]]
- [[reliability]]
- [[caching]]