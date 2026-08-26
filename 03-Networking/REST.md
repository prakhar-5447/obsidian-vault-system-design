# REST (Representational State Transfer)

## 1. Definition

REST is an **architectural style** for designing networked APIs (not a protocol) — typically implemented over [[http]]. It models everything as **resources**, identified by URLs, manipulated via standard HTTP methods.

Coined by Roy Fielding in his 2000 dissertation, describing the architectural style behind the web itself.

---

## 2. REST Constraints (What Makes an API "RESTful")

|Constraint|Meaning|
|---|---|
|**Client-server**|Separation of concerns — client and server evolve independently|
|**Statelessness**|Each request contains all info needed; server holds no client session state|
|**Cacheability**|Responses must define themselves as cacheable or not (`Cache-Control`)|
|**Uniform interface**|Consistent resource identification (URLs), standard methods (GET/POST/etc.), self-descriptive messages|
|**Layered system**|Client can't tell if it's talking directly to the server or through intermediaries (proxies, LBs, CDNs)|
|**Code on demand** (optional)|Server can send executable code (e.g. JavaScript) — rarely used in API context|

Most "REST APIs" in practice are **RESTful/REST-like** and don't strictly satisfy HATEOAS (below) — worth knowing the distinction.

---

## 3. Resource-Oriented URL Design

|Good|Bad|
|---|---|
|`GET /users/123`|`GET /getUser?id=123`|
|`POST /users`|`POST /createUser`|
|`GET /users/123/orders`|`GET /getOrdersForUser?id=123`|
|`DELETE /users/123`|`POST /deleteUser?id=123`|

**Principle:** URLs identify **nouns** (resources); HTTP methods express the **verb** (action). Nesting (`/users/123/orders`) expresses relationships.

---

## 4. HATEOAS (Hypermedia as the Engine of Application State)

The "purist" REST constraint most real APIs skip: responses should include **links** to related actions/resources, so clients discover the API dynamically instead of hardcoding URLs.

```json
{
  "id": 123,
  "name": "Prakhar",
  "links": [
    { "rel": "self", "href": "/users/123" },
    { "rel": "orders", "href": "/users/123/orders" }
  ]
}
```

Rare in practice (most APIs are documented + hardcoded client-side instead) but a good interview differentiator to mention and explain why it's often skipped (added complexity for limited real-world benefit at most companies' scale).

---

## 5. REST API Design Best Practices

|Practice|Why|
|---|---|
|Use plural nouns (`/users` not `/user`)|Consistency|
|Version your API (`/v1/users`)|Avoid breaking existing clients on changes|
|Use proper status codes|Let clients handle errors programmatically, not by parsing text|
|Support pagination (`?page=2&limit=50` or cursor-based)|Avoid huge payloads, essential at scale|
|Filtering/sorting via query params (`?status=active&sort=-created_at`)|Keeps URLs resource-focused|
|Idempotent `PUT`/`DELETE`|Enables safe client retries (see [[reliability]])|
|Rate limiting + `429` responses|Protect backend from abuse/overload|

**Pagination strategies:**

- **Offset-based** (`?page=2&limit=50`) — simple, but breaks under concurrent inserts (items shift between pages)
- **Cursor-based** (`?after=<opaque_token>`) — stable under concurrent writes, preferred at scale (used by Twitter/Facebook APIs)

---

## 6. REST vs gRPC vs GraphQL (Quick Orientation)

| |REST|[[grpc]]|GraphQL|
|---|---|---|---|
|Format|JSON (usually)|Protobuf (binary)|JSON|
|Contract|Loose/documented (OpenAPI optional)|Strict (`.proto` schema)|Strict (GraphQL schema)|
|Over-fetching|Common (fixed response shape)|N/A (defined per RPC)|Solved (client asks exactly what it needs)|
|Best for|Public APIs, simplicity, broad compatibility|Internal service-to-service, high performance|Client-driven flexible queries (mobile apps, complex UIs)|

Full gRPC comparison in [[grpc]].

---

## 7. Interview Talking Points

- Emphasize **statelessness** as the key enabler of horizontal scaling for REST APIs (ties to [[scalability]]).
- Bring up **idempotency** when discussing retry-safety and client error handling.
- When asked "REST vs gRPC vs GraphQL," give the actual decision criteria (public vs internal API, over-fetching concerns, performance needs) rather than just naming the options.
- Mention versioning strategy (URL path vs header-based) as a real design decision with trade-offs (URL versioning is simpler/more visible; header versioning is "cleaner" but harder to debug/test in browser).

---

## 8. Resources

- [Roy Fielding's Dissertation – Ch. 5 (REST)](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm) — primary source
- [Microsoft REST API Guidelines (GitHub)](https://github.com/microsoft/api-guidelines)
- [Stripe API Docs](https://stripe.com/docs/api) — widely cited as a REST API design gold standard
- [REST API Tutorial](https://restfulapi.net/)

---

## Related Notes

- [[http]]
- [[grpc]]
- [[scalability]]
- [[reliability]]
- [[system-design-networking-index]]