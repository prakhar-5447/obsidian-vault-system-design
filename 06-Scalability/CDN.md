# CDN (Content Delivery Network)

## 1. Definition

A CDN is a **geographically distributed network of edge servers (PoPs — Points of Presence)** that caches and serves content **physically close to the end user**, reducing [[latency]] and offloading traffic from origin servers.

Conceptually the CDN is a caching layer (see [[caching]]) placed at the network edge, plus request routing (often via [[dns]] — GeoDNS/anycast) to send users to their nearest PoP.

---

## 2. How a CDN Request Flows

```
User (Mumbai) → DNS resolves to nearest PoP (Mumbai edge server)
   ↓
Edge server has content cached? 
   YES → serve directly (fast, no origin trip)
   NO  → "cache miss" → fetch from origin server (e.g. US datacenter) → cache it → serve to user
```

First request to a region = **cache miss**, slower (full round trip to origin). Subsequent requests from nearby users = **cache hit**, served from the edge — much faster.

---

## 3. What CDNs Cache

|Content type|Typical cacheability|
|---|---|
|**Static assets** (images, CSS, JS, videos, fonts)|Highly cacheable — rarely change, safe to cache long|
|**API responses**|Sometimes — only if safe to be slightly stale (e.g. public product listings), controlled via `Cache-Control` headers|
|**Personalized/dynamic content**|Usually NOT cached (or cached per-user, rarely worth it) — user-specific dashboards, account pages|
|**HTML pages**|Increasingly cached too, especially with "edge rendering"/ISR patterns in modern frameworks|

---

## 4. Push vs Pull CDN

| |Pull CDN|Push CDN|
|---|---|---|
|How content gets to edge|CDN fetches from origin **on first request** (lazy)|You **proactively upload** content to the CDN ahead of time|
|Best for|General web traffic, unpredictable access patterns|Large media files, known release schedules (e.g. game patch releases)|
|Staleness risk|First request per region is slow; relies on TTL/invalidation for freshness|You control exactly what's live and when|

Most CDNs (Cloudflare, CloudFront, Fastly) default to **pull** — simpler to operate.

---

## 5. Cache Invalidation at the CDN Layer

Same core problem as [[cache-invalidation]], applied at the edge:

- **TTL-based** (`Cache-Control: max-age=3600`) — simplest, most common
- **Explicit purge/invalidation API** — force-remove a specific URL from all edge caches immediately after a deploy/content update
- **Versioned/hashed asset URLs** (`app.a1b2c3.js`) — sidesteps invalidation entirely (same technique described in [[cache-invalidation]])
- **`stale-while-revalidate`** — serve the cached version instantly while refreshing in the background (also covered in [[cache-stampede]])

---

## 6. CDN Benefits Beyond Latency

|Benefit|How|
|---|---|
|**Reduced origin load**|Most requests never reach your servers at all|
|**DDoS resilience**|Attack traffic absorbed/distributed across many PoPs rather than hitting one origin (ties to [[dns]]'s anycast section)|
|**Higher availability**|If origin briefly goes down, CDN can often still serve cached content|
|**TLS termination at the edge**|Reduces handshake RTT for the user (see [[https]]) — full TLS handshake happens with the nearby edge, not a distant origin|
|**Bandwidth cost savings**|Origin serves far less traffic, which is often billed|

---

## 7. Modern CDN Capabilities (Beyond Static Caching)

- **Edge compute** (Cloudflare Workers, Lambda@Edge) — run application logic at the edge, not just cache lookup (e.g. A/B testing, auth checks, request rewriting close to the user)
- **Image optimization** — on-the-fly resizing/format conversion (WebP/AVIF) per-device at the edge
- **DDoS/WAF (Web Application Firewall)** — security filtering happens before traffic ever reaches origin

---

## 8. Interview Talking Points

- Bring up CDN whenever a design serves **static assets or globally distributed users** — it's usually a "free win" interviewers expect you to mention early.
- Explicitly connect CDN's request routing to [[dns]] (GeoDNS/anycast) and its caching mechanics to [[caching]]/[[cache-invalidation]] — shows you see it as an instance of concepts you already know, not a separate black box.
- Mention **cache-control headers as the actual API** for controlling CDN behavior — a concrete, correct detail interviewers like to hear.
- For dynamic/personalized content, be clear about what should and shouldn't be CDN-cached — over-claiming "just CDN everything" is a common mistake.

---

## 9. Resources

- [Cloudflare Learning – What is a CDN?](https://www.cloudflare.com/learning/cdn/what-is-a-cdn/)
- [AWS CloudFront Docs](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html)
- [MDN – HTTP Caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [High Performance Browser Networking – Ch. 5 (Primer on Web Performance)](https://hpbn.co/primer-on-web-performance/)

---

## Related Notes

- [[caching]]
- [[cache-invalidation]]
- [[dns]]
- [[latency]]
- [[load-balancing]]