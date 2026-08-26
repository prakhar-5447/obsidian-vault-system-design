# HTTP (HyperText Transfer Protocol)

## 1. Definition

HTTP is an **application-layer** (OSI Layer 7) **request-response** protocol used for communication between clients (browsers, apps) and servers. It's **stateless** — each request is independent, with no memory of previous requests unless state is explicitly carried (cookies, tokens, sessions).

Runs over TCP (HTTP/1.1, HTTP/2) or UDP-based QUIC (HTTP/3) — see [[tcp]], [[udp]].

---

## 2. Request/Response Structure

**Request:**

```
GET /api/users/123 HTTP/1.1
Host: example.com
Authorization: Bearer <token>
Accept: application/json

(optional body)
```

**Response:**

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 128

{"id": 123, "name": "Prakhar"}
```

---

## 3. HTTP Methods

| Method    | Purpose                                           | Idempotent?  | Safe (no side effects)? |
| --------- | ------------------------------------------------- | ------------ | ----------------------- |
| `GET`     | Retrieve resource                                 | Yes          | Yes                     |
| `POST`    | Create resource / non-idempotent action           | No           | No                      |
| `PUT`     | Replace resource entirely                         | Yes          | No                      |
| `PATCH`   | Partial update                                    | No (usually) | No                      |
| `DELETE`  | Remove resource                                   | Yes          | No                      |
| `HEAD`    | Like GET but headers only                         | Yes          | Yes                     |
| `OPTIONS` | Discover allowed methods (used in CORS preflight) | Yes          | Yes                     |

**Idempotent** = calling it multiple times has the same effect as calling it once (important for safe retries — see [[reliability]]).

---

## 4. Status Codes (Know the Groups)

| Range | Meaning       | Common codes                                                                             |
| ----- | ------------- | ---------------------------------------------------------------------------------------- |
| 1xx   | Informational | 101 Switching Protocols (used in WebSocket upgrade)                                      |
| 2xx   | Success       | 200 OK, 201 Created, 204 No Content                                                      |
| 3xx   | Redirection   | 301 Moved Permanently, 302 Found, 304 Not Modified                                       |
| 4xx   | Client error  | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests   |
| 5xx   | Server error  | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout |

---

## 5. HTTP/1.1 vs HTTP/2 vs HTTP/3

|                       | HTTP/1.1                                                                            | HTTP/2                                              | HTTP/3                                                   |
| --------------------- | ----------------------------------------------------------------------------------- | --------------------------------------------------- | -------------------------------------------------------- |
| Transport             | TCP                                                                                 | TCP                                                 | QUIC (UDP-based)                                         |
| Multiplexing          | No (1 request per connection at a time; workaround = multiple parallel connections) | Yes (multiple streams over 1 connection)            | Yes (streams are independent — no TCP HOL blocking)      |
| Head-of-line blocking | Yes, badly                                                                          | Reduced app-level, but still TCP-level HOL blocking | Solved — QUIC streams don't block each other             |
| Header compression    | No                                                                                  | Yes (HPACK)                                         | Yes (QPACK)                                              |
| Server push           | No                                                                                  | Yes (rarely used in practice)                       | Yes                                                      |
| Handshake cost        | TCP handshake + TLS handshake (2-3 RTT)                                             | Same as HTTP/1.1                                    | 0-RTT or 1-RTT (QUIC combines transport + TLS handshake) |
| Encryption            | Optional                                                                            | De facto required (browsers mandate TLS for HTTP/2) | Mandatory, built into QUIC                               |

**Interview point:** HTTP/2's multiplexing helps at the _application_ layer, but since it still runs on one TCP connection, a single lost packet stalls _all_ streams (TCP-level HOL blocking) — this is exactly what HTTP/3/QUIC fixes by using UDP with per-stream reliability.

---

## 6. Statelessness & How State Is Added Back

HTTP itself has no memory between requests. State is layered on top via:

- **Cookies** — client stores a token, sent with every request
- **Sessions** — server-side state keyed by a session ID (in cookie)
- **Tokens (JWT, OAuth)** — self-contained, signed state carried by client
- **Caching headers** (`ETag`, `Cache-Control`, `Last-Modified`) — enable conditional requests (304 Not Modified)

---

## 7. Caching Headers (System Design Relevant)

|Header|Purpose|
|---|---|
|`Cache-Control: max-age=3600`|How long a response can be cached|
|`ETag`|Fingerprint of resource; client sends `If-None-Match` to check staleness|
|`Last-Modified`|Timestamp-based staleness check|
|`Cache-Control: no-cache` / `no-store`|Prevent caching (sensitive data)|

Interacts directly with [[latency]] and CDN design.

---

## 8. Interview Talking Points

- Know when to reach for HTTP/2 or HTTP/3 benefits (many small concurrent requests, mobile/high-latency networks) vs when HTTP/1.1 + keep-alive is sufficient.
- Understand idempotency (`PUT`/`DELETE` vs `POST`) — relevant when designing safe retry logic for clients (ties to [[reliability]]).
- Statelessness is _why_ HTTP servers scale horizontally so easily — no server affinity needed unless you introduce sticky sessions (ties to [[scalability]]).

---

## 9. Resources

- [MDN – HTTP Overview](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview)
- [High Performance Browser Networking – Ch. 9-13 (HTTP)](https://hpbn.co/http2/)
- [Cloudflare – What is HTTP/3?](https://www.cloudflare.com/learning/performance/what-is-http3/)
- [RFC 9110 – HTTP Semantics (current spec)](https://www.rfc-editor.org/rfc/rfc9110)

---

## Related Notes

- [[https]]
- [[rest]]
- [[tcp]]
- [[udp]]
- [[websocket]]
- [[grpc]]
- [[system-design-networking-index]]