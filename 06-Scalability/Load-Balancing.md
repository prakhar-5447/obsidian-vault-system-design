# Load Balancing

## 1. Definition

A load balancer distributes incoming traffic across **multiple backend servers**, so no single server is overwhelmed. Core enabler of both [[scalability]] (horizontal scaling only works if traffic is spread) and [[availability]] (routes around unhealthy nodes).

---

## 2. Layer 4 vs Layer 7 Load Balancing

| |L4 (Transport layer)|L7 (Application layer)|
|---|---|---|
|Operates on|IP + TCP/UDP port|HTTP headers, URL path, cookies, content|
|Speed|Faster (less inspection)|Slower (parses the request)|
|Routing intelligence|Basic (IP/port hashing, round robin)|Smart (route `/api/*` to service A, `/static/*` to CDN, route by header/cookie)|
|SSL/TLS termination|Usually passes through|Can terminate TLS at the LB|
|Examples|AWS NLB, raw TCP load balancers|AWS ALB, Nginx, HAProxy, Envoy|

**Interview point:** L7 is the default choice for most web/API architectures today because content-aware routing (path-based, header-based) is usually needed — L4 is reserved for raw throughput-critical or non-HTTP traffic.

---

## 3. Load Balancing Algorithms

|Algorithm|How it works|Best for|
|---|---|---|
|**Round robin**|Requests distributed sequentially across servers|Uniform servers, uniform request cost|
|**Weighted round robin**|Like round robin, but servers with more capacity get proportionally more requests|Heterogeneous server sizes|
|**Least connections**|Route to the server with fewest active connections|Variable request duration (long-lived connections)|
|**Least response time**|Route to the server responding fastest currently|Latency-sensitive workloads|
|**IP hash**|Hash client IP → consistently routes same client to same server|Session affinity without cookies|
|**Random**|Pick a server at random|Simple, surprisingly effective at scale|

---

## 4. Health Checks

The load balancer periodically probes backend servers (`GET /health`) and stops routing to any that fail — this is what actually makes redundancy useful for [[availability]]. Two failure-detection knobs to always mention: **unhealthy threshold** (consecutive failures before marking down) and **healthy threshold** (consecutive successes before marking back up) — tuning these trades off false-positive removals against slow failure detection.

---

## 5. Sticky Sessions

Some architectures need a client to consistently reach the **same backend server** (e.g. WebSocket connections, in-memory session state not externalized to Redis). Implemented via:

- Cookie-based affinity (LB sets a cookie identifying the assigned server)
- IP hash

**Trade-off:** sticky sessions undermine even load distribution and complicate failover (if that server dies, the client's session/state is gone unless externalized) — prefer **stateless services + external session store** ([[redis]]) over sticky sessions where possible. Real exception: [[websocket]] connections are inherently stateful and often need this.

---

## 6. DNS-Based Load Balancing vs Dedicated Load Balancer

DNS round-robin ([[dns]] section 6) is coarse — no real-time health awareness, bound by TTL. A dedicated LB (software or hardware) operates per-request, with live health checks and fine-grained algorithms. **Real systems layer both**: DNS/GeoDNS routes traffic to the nearest **region**, then a regional load balancer distributes across servers **within** that region.

---

## 7. Load Balancer as a Single Point of Failure

The LB itself must be highly available:

- **Active-passive**: standby LB takes over on failure (via floating IP / VRRP)
- **Active-active**: multiple LBs behind DNS or anycast, all serving traffic simultaneously
- Cloud-managed LBs (AWS ALB/NLB, GCP Load Balancer) handle this transparently — worth mentioning you'd default to a managed offering in most real designs rather than hand-rolling HA for a self-hosted LB.

---

## 8. Interview Talking Points

- Specify L4 vs L7 explicitly based on the routing intelligence actually needed.
- Always mention **health checks** as the mechanism that makes redundancy real — a system with 5 replicas but no health checks doesn't actually route around failures.
- Prefer **stateless services** over sticky sessions when discussing scaling — call out WebSocket as the legitimate exception (ties to [[websocket]]'s scaling challenges).
- Mention that the LB itself needs an HA story — don't leave it as an unexamined SPOF in your diagram.

---

## 9. Resources

- [NGINX – Load Balancing Guide](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
- [AWS – Elastic Load Balancing types](https://aws.amazon.com/elasticloadbalancing/)
- [HAProxy Docs](http://www.haproxy.org/#docs)
- [System Design Primer – Load Balancer section](https://github.com/donnemartin/system-design-primer#load-balancer)

---

## Related Notes

- [[scalability]]
- [[availability]]
- [[dns]]
- [[websocket]]
- [[cdn]]