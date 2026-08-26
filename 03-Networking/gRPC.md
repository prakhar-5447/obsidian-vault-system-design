# gRPC

## 1. Definition

gRPC is a **high-performance RPC (Remote Procedure Call) framework** built by Google, running over **HTTP/2**, using **Protocol Buffers (protobuf)** as its default serialization format. You call a remote method as if it were local — the framework handles serialization + network transport.

Widely used for **internal service-to-service communication** in microservice architectures.

---

## 2. Core Concepts

- **`.proto` file**: defines the service contract (methods + message types) in a language-agnostic schema
- **Code generation**: protobuf compiler generates client & server stubs in any supported language (Go, Java, Python, etc.) from the `.proto` file
- **Protobuf**: binary serialization format — smaller and faster to (de)serialize than JSON

```protobuf
service UserService {
  rpc GetUser (GetUserRequest) returns (User);
}

message GetUserRequest {
  int32 id = 1;
}

message User {
  int32 id = 1;
  string name = 2;
}
```

---

## 3. Four Types of gRPC Calls (Leverages HTTP/2 Streaming)

|Type|Description|Example use case|
|---|---|---|
|**Unary**|One request → one response (like a normal REST call)|`GetUser(id)`|
|**Server streaming**|One request → stream of responses|Live price feed, log tailing|
|**Client streaming**|Stream of requests → one response|Uploading a large file in chunks|
|**Bidirectional streaming**|Both sides stream independently|Real-time chat, live collaboration|

This is only possible because gRPC runs on **HTTP/2**, which supports multiplexed, long-lived streams over a single connection — see [[http]].

---

## 4. gRPC vs REST — Detailed Comparison

|                   | gRPC                                                           | REST                                           |
| ----------------- | -------------------------------------------------------------- | ---------------------------------------------- |
| Transport         | HTTP/2 (mandatory)                                             | HTTP/1.1 or HTTP/2                             |
| Payload format    | Protobuf (binary)                                              | JSON (text, human-readable)                    |
| Payload size      | Smaller (binary encoding)                                      | Larger (text overhead)                         |
| Speed             | Faster (binary + multiplexing + no repeated headers via HPACK) | Slower relatively                              |
| Contract          | Strict schema (`.proto`), strongly typed                       | Loose (OpenAPI optional, often undocumented)   |
| Streaming         | Native (4 modes)                                               | Not native (needs WebSocket/SSE workaround)    |
| Browser support   | Poor (needs gRPC-Web proxy layer)                              | Native (universal)                             |
| Human readability | No (binary, needs tooling to inspect)                          | Yes (plain JSON, curl-friendly)                |
| Best for          | Internal microservice-to-microservice calls                    | Public-facing APIs, broad client compatibility |
|                   |                                                                |                                                |

**Interview rule of thumb:** gRPC for **internal, performance-sensitive, polyglot microservice** communication. REST/JSON for **public APIs** where broad compatibility, debuggability, and human-readability matter more than raw performance.

---

## 5. Why gRPC Is Faster

1. **Binary protobuf** — smaller payloads, faster parse than JSON text parsing
2. **HTTP/2 multiplexing** — many concurrent RPCs over one TCP connection, no head-of-line blocking at the app layer
3. **HPACK header compression** — avoids repeating headers on every call
4. **Strict typing** — no runtime schema validation/guessing overhead like dynamically-typed JSON parsing

---

## 6. Trade-offs / Limitations

- **Not browser-native** — needs `grpc-web` + a proxy (e.g. Envoy) to translate for browser clients, adding complexity
- **Harder to debug** — can't just `curl` and read the response; need tools like `grpcurl` or `BloomRPC`
- **Tighter coupling** — schema changes require regenerating client/server code (though protobuf has good backward-compatibility rules: don't reuse field numbers, mark deprecated fields, etc.)
- **Less human-readable** — harder for a new engineer to inspect traffic ad hoc vs plain JSON

---

## 7. Interview Talking Points

- Bring up gRPC when designing **internal microservice communication** in a system design interview, especially for performance-critical, high-QPS paths.
- Mention **streaming RPCs** as a natural fit for real-time internal data flows (e.g. service A streaming live inventory updates to service B) — an alternative to Kafka/message queues for point-to-point streaming.
- Explain the trade-off honestly: gRPC isn't strictly "better" than REST — it optimizes for different priorities (performance + type safety vs simplicity + compatibility).

---

## 8. Resources

- [gRPC Official Docs](https://grpc.io/docs/what-is-grpc/introduction/)
- [Protocol Buffers Docs](https://protobuf.dev/overview/)
- [gRPC vs REST – Google Cloud blog comparison](https://cloud.google.com/blog/products/api-management/understanding-grpc-openapi-and-rest-and-when-to-use-them)
- [grpc-web (browser support)](https://github.com/grpc/grpc-web)

---

## Related Notes

- [[http]]
- [[rest]]
- [[websocket]]
- [[system-design-networking-index]]