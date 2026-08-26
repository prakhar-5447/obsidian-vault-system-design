# Idempotency

## 1. Definition

An operation is **idempotent** if performing it multiple times has the **same effect** as performing it once. Critical for building safe retries into any distributed system — since networks are unreliable, a client can never truly know if a request failed before or after the server processed it.

```
"Set balance to $100"     → idempotent (repeat = still $100)
"Add $100 to balance"     → NOT idempotent (repeat = $100 more each time)
```

---

## 2. Why It Matters: The Retry Ambiguity Problem

```
Client → sends "ChargeCard $50" → [request succeeds on server, but response is lost on the way back]
Client → times out, never got a response → doesn't know if the charge happened
Client → retries "ChargeCard $50" → if not idempotent, customer is charged TWICE
```

This is not a rare edge case — it's the **default behavior of any network call under timeout/retry**, which is why every system that retries (and most should — see [[message-queues]], [[reliability]]) must design for idempotency at the operations that matter.

---

## 3. HTTP Methods and Idempotency (Baseline)

Covered briefly in [[http]] — worth restating precisely here:

|Method|Idempotent?|Why|
|---|---|---|
|`GET`|Yes|Read-only, no side effect|
|`PUT`|Yes|Replaces the resource entirely — same input, same result every time|
|`DELETE`|Yes|Resource is gone either way (first call deletes it, repeat calls are no-ops)|
|`POST`|**No**|Typically "create a new thing" — repeating creates duplicates, unless explicitly made idempotent (see below)|
|`PATCH`|Usually not|Partial update — depends on the operation (`set field = X` is idempotent; `increment field by 1` is not)|

**Interview trap:** `POST` isn't inherently non-idempotent by law — it's just the _convention_ that it usually represents "create," which isn't naturally idempotent. You can and often should make specific `POST` endpoints idempotent explicitly (see below).

---

## 4. How to Make Non-Idempotent Operations Idempotent

### Idempotency Keys (Most Common Pattern)

Client generates a unique key (UUID) per **logical** operation attempt and sends it with the request. Server stores completed keys and their results; if the same key arrives again, it returns the **stored result** instead of re-executing.

```
POST /charge
Idempotency-Key: 7d8f3e21-...
{ "amount": 50 }
```

```python
def charge(idempotency_key, amount):
    existing = db.get(f"idempotency:{idempotency_key}")
    if existing:
        return existing.result   # already processed — return cached response, don't re-charge

    result = actually_charge_card(amount)
    db.set(f"idempotency:{idempotency_key}", result, ttl=86400)
    return result
```

**Used by:** Stripe, PayPal, most payment APIs — the industry-standard way to make `POST` safe to retry.

**Critical detail:** the check-and-store must be **atomic** (e.g. a single DB transaction, or a Redis `SET NX`) — otherwise two concurrent retries with the same key can both pass the "does it exist" check before either writes the result, causing a double-charge race (this is the same read-modify-write race discussed in [[cache-invalidation]]).

### Conditional Writes (Compare-and-Swap)

```sql
UPDATE inventory SET count = count - 1, version = version + 1
WHERE product_id = 5 AND version = 3;
```

If another process already changed the version, this update affects 0 rows — the client detects this and can safely retry with fresh data, rather than blindly applying the same delta twice. This is optimistic concurrency control (see [[transactions]]).

### Natural Idempotency via Design

Sometimes you can restructure the operation itself to be naturally idempotent:

```
Not idempotent: "increment counter by 1"
Idempotent:     "set counter to 42" (an absolute value, computed client-side, sent instead of a delta)
```

Or, for creation: use a **client-generated ID** instead of letting the server generate one — `PUT /orders/{client_generated_uuid}` instead of `POST /orders` — turns "create" into "create-or-replace," which is naturally idempotent.

---

## 5. Idempotency in Message Consumers

At-least-once delivery (the realistic guarantee for most queues, see [[message-queues]] section 4) means a consumer **will** occasionally process the same message twice. The consumer itself must be idempotent:

```python
def process_message(event):
    if db.exists(f"processed:{event.id}"):
        return  # already handled, skip
    do_the_actual_work(event)
    db.set(f"processed:{event.id}", True, ttl=...)
```

Same pattern as the API idempotency key — dedupe by a unique event/message ID, store a marker once processed.

---

## 6. Idempotency vs Exactly-Once Delivery

**Idempotency is what makes "at-least-once + retries" behave like "exactly-once" from the caller's perspective** — without needing the messaging system itself to guarantee true exactly-once delivery (which is very hard/expensive in distributed systems, see [[message-queues]]). This is the practical, widely-used answer whenever "exactly-once" comes up in an interview.

---

## 7. Interview Talking Points

- Whenever discussing retries, timeouts (see [[retry-and-timeout]]), or payment/money-moving flows, bring up idempotency **proactively** — it's one of the most consistently expected details in system design interviews.
- Explain the **idempotency key pattern** concretely (client-generated key, server stores result, atomic check-and-store) rather than just naming it.
- Mention that the check-and-store step must be **atomic** — this is the detail that separates a surface-level answer from a correct one.
- Connect explicitly to consumer-side idempotency when discussing [[message-queues]] or [[event-driven-architecture]] — "idempotent consumer" is the expected answer to "what if a message is delivered twice?"

---

## 8. Resources

- [Stripe Docs – Idempotent Requests](https://stripe.com/docs/api/idempotent_requests)
- [AWS – Making retries safe with idempotency](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)
- _Designing Data-Intensive Applications_ by Martin Kleppmann — Ch. 11 (Idempotence in stream processing)
- [Stripe Engineering Blog – Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency)

---

## Related Notes

- [[reliability]]
- [[message-queues]]
- [[retry-and-timeout]]
- [[transactions]]
- [[http]]
- [[event-driven-architecture]]