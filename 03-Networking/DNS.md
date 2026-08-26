## 1. What is DNS?

DNS translates human-readable **domain names** (e.g. `api.example.com`) into machine-usable **IP addresses** (e.g. `142.250.183.14`). It's a distributed, hierarchical, globally replicated key-value lookup system — often called "the phonebook of the internet."

**Why it matters in system design:**

- First hop in nearly every client request → latency here affects everything downstream
- Used for load balancing, failover, geo-routing (not just name resolution)
- A single point of failure if misconfigured (see: many major outages traced to DNS)

---
## 2. DNS Hierarchy

```
                    "." (root)
                     |
      -------------------------------
      |              |              |
    .com           .org           .in     (TLD - Top Level Domain)
      |
  example.com                              (Authoritative domain)
      |
  api.example.com                          (Subdomain)
```

|Level|Example|Managed by|
|---|---|---|
|Root|`.`|ICANN (13 root server clusters, replicated via anycast)|
|TLD|`.com`, `.org`, `.in`|Registry operators (Verisign for `.com`)|
|Authoritative|`example.com`|Domain owner's DNS provider (Route53, Cloudflare)|
|Subdomain|`api.example.com`|Same or delegated provider|

---

## 3. Resolution Flow (Full Path)

1. **Browser cache** → checked first (TTL-bound)
2. **OS cache** (stub resolver)
3. **Recursive resolver** (ISP or public: 8.8.8.8, 1.1.1.1) — does the heavy lifting
4. Resolver queries **Root server** → returns TLD server address
5. Resolver queries **TLD server** → returns Authoritative NS address
6. Resolver queries **Authoritative NS** → returns the actual IP (A/AAAA record)
7. Resolver **caches** result (per TTL) and returns to client

```
Client → Recursive Resolver → Root NS → TLD NS → Authoritative NS
                ↑___________________________________|
                     (final answer cached & returned)
```

**Key insight for interviews:** most lookups are cache hits at the recursive resolver layer — root/TLD servers are rarely hit directly per-request, which is why DNS scales globally despite serving billions of queries/sec.

---

## 4. Record Types (Know These)

| Record  | Purpose                                      | Example                                        |
| ------- | -------------------------------------------- | ---------------------------------------------- |
| `A`     | Hostname → IPv4                              | `example.com → 93.184.216.34`                  |
| `AAAA`  | Hostname → IPv6                              | `example.com → 2606:2800:220:1::`              |
| `CNAME` | Alias → another hostname                     | `www.example.com → example.com`                |
| `MX`    | Mail server routing                          | `example.com → mail.example.com (priority 10)` |
| `NS`    | Delegates a subdomain to nameservers         | `example.com → ns1.provider.com`               |
| `TXT`   | Arbitrary text (SPF, domain verification)    | `v=spf1 include:_spf.google.com ~all`          |
| `SOA`   | Zone metadata (admin, refresh, TTL defaults) | —                                              |
| `PTR`   | Reverse lookup: IP → hostname                | Used in email anti-spam checks                 |
| `SRV`   | Service location (host + port)               | Used by SIP, XMPP, etc.                        |

⚠️ **Common gotcha:** `CNAME` cannot coexist with other records on the same name, and can't be used at the zone apex (root domain) in most providers — this is why `ALIAS`/`ANAME` (Route53 calls it "Alias record") exists as a workaround.

---

## 5. Caching & TTL

- **TTL (Time To Live)**: how long a record can be cached, in seconds, set per-record.
- Trade-off:
    - **Low TTL** (e.g. 60s) → faster failover/updates propagate quickly, but more DNS query load & latency
    - **High TTL** (e.g. 86400s / 1 day) → less load, but slow to propagate changes (bad during incident response / migrations)
- **Best practice:** lower TTL _before_ a planned migration (e.g. to 60s, wait for old TTL to expire, then cut over) so the change propagates fast.

---

## 6. DNS-Based Load Balancing & Traffic Routing

This is the system-design-relevant part — DNS isn't just "lookup," it's a routing mechanism:

| Strategy                  | How it works                                                               | Use case                                 |
| ------------------------- | -------------------------------------------------------------------------- | ---------------------------------------- |
| **Round-robin DNS**       | Multiple `A` records for one name, resolver returns them in rotating order | Simple load spreading (no health checks) |
| **GeoDNS**                | Returns different IPs based on the resolver's geographic location          | Route users to nearest region/datacenter |
| **Latency-based routing** | Returns IP of the lowest-latency endpoint (AWS Route53 does this)          | Multi-region low-latency apps            |
| **Weighted routing**      | Distributes traffic by configured percentage                               | Canary releases, A/B testing             |
| **Failover routing**      | Health-checks endpoints, routes away from unhealthy ones                   | High availability                        |

**Limitation to mention in interviews:** DNS-based LB is coarse-grained — it can't react instantly (bound by TTL + client-side caching that ignores TTL), and it doesn't see real-time backend load like an L4/L7 load balancer (e.g. ALB, Envoy) does. Real systems layer DNS routing (coarse, geo/region-level) **on top of** a proper load balancer (fine-grained, per-request).

---

## 7. Anycast

Multiple physical servers **share the same IP address**; BGP routing sends the client to the topologically nearest one.

- Used by: root DNS servers, Cloudflare/Google public resolvers, CDNs
- Benefit: built-in geographic distribution + DDoS resilience (attack traffic gets spread across many PoPs) (**Point of Presence**)
- Different from GeoDNS: anycast is routing-layer (same IP, network picks the path); GeoDNS is DNS-layer (different IPs returned based on location)

---

## 8. DNS & System Design Trade-offs (Interview Talking Points)

- **DNS as a bottleneck:** if your resolver/provider goes down, your whole service becomes unreachable even if servers are healthy. Mitigate with multiple DNS providers (e.g. Route53 + Cloudflare) or a robust anycast-based provider.
- **Propagation delay:** DNS is _eventually consistent_. Never assume instant failover — plan TTL strategy ahead of migrations/incidents.
- **DNS as service discovery:** internally, systems often use DNS-based service discovery (e.g. Kubernetes CoreDNS, Consul) instead of a hardcoded config — pods/services get resolvable names, enabling dynamic scaling.
- **Security concerns:**
    - **DNS spoofing/cache poisoning** → mitigated by DNSSEC (cryptographically signs records)
    - **DDoS on DNS** → mitigated by anycast + rate limiting
    - **DNS over HTTPS/TLS (DoH/DoT)** → encrypts queries between client and resolver (privacy, prevents MITM/snooping) (**Man-in-the-Middle attac**)

---

## 9. Quick Recap Table

|Concept|One-liner|
|---|---|
|Recursive resolver|Does the multi-step lookup on client's behalf|
|Authoritative NS|Source of truth for a domain's records|
|TTL|Cache duration; tune before migrations|
|CNAME vs A|CNAME → hostname alias; A → direct IP|
|GeoDNS|Different IP per region|
|Anycast|Same IP, nearest physical server via BGP|
|DNSSEC|Prevents spoofing via signed records|

---

## 10. Resources

- [Cloudflare Learning: What is DNS?](https://www.cloudflare.com/learning/dns/what-is-dns/) — best conceptual intro
- [How DNS Works (comic)](https://howdns.works/) — visual, intuitive explainer
- [AWS Route53 Routing Policies Docs](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html) — real-world routing policy reference
- [RFC 1034 – Domain Names, Concepts & Facilities](https://www.rfc-editor.org/rfc/rfc1034) — primary source, skim only
- _Designing Data-Intensive Applications_ — Ch. on system communication (not DNS-specific but good context)
- [ByteByteGo – DNS system design videos](https://bytebytego.com/) — good for interview-framing

---

## Related Notes

- [[load-balancing]]
- [[cdn]]
- [[tcp-ip-basics]]
- [[system-design-networking-index]]