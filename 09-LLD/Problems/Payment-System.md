# Payment System — Low Level Design

## 1. Requirements

### Functional

- Merchant/client initiates a payment request (amount, currency, payment method details)
- System supports multiple payment methods: Credit/Debit Card, Wallet, UPI/Bank Transfer, Buy-Now-Pay-Later
- System routes the payment to the correct downstream payment provider/processor for that method
- System supports refunds (full or partial) against a previously successful payment
- System records every payment attempt and its outcome for audit/reconciliation
- Client can query the current status of a payment (Pending, Success, Failed, Refunded)
- System supports **idempotent** payment requests — a retried request with the same idempotency key must not double-charge
- System handles asynchronous confirmation via webhooks from external providers (some payment methods don't confirm synchronously)
- System supports multiple currencies and basic currency conversion at the point of charge (optional/extensible)

### Non-Functional

- **Reliability** — a payment must never be lost, and never double-charged, even under network failures/retries (the defining requirement — ties to [[reliability]], [[idempotency]])
- **Consistency** — payment status must be unambiguous and consistent across the system, the merchant's view, and the actual state at the payment provider (ties to [[consistency]], [[transactions]])
- **Availability** — the system should degrade gracefully if one provider is down (fail over to another, or fail fast with a clear error) rather than hanging (ties to [[availability]], [[retry]], [[timeout]])
- **Security** — sensitive payment details (card numbers, CVV) must never be stored in plaintext; PCI-DSS-style tokenization should be used instead
- **Auditability** — every state transition of every payment must be logged immutably for reconciliation and dispute handling
- **Extensibility** — adding a new payment method or provider should not require changes to the core payment orchestration logic
- **Scalability** — must handle high transaction volume with low latency, especially during peak shopping periods (ties to [[scalability]], [[throughput]])

---

## 2. Actors

- **Merchant / Client Application** — initiates payment requests, queries status, requests refunds
- **Customer** — the person whose payment method is being charged (mostly opaque to this system; represented via the payment method details/token)
- **Payment Provider (external)** — the actual processor (Visa/Mastercard networks, a bank, a wallet provider) that moves real money
- **Webhook Listener** (system-internal actor) — receives asynchronous status updates from external providers

---

## 3. Use Cases

- Initiate a payment (charge) via a specified method
- Route the payment to the correct provider integration for that method
- Handle a provider's synchronous response (success/failure) or asynchronous webhook confirmation
- Retry a transient provider failure safely (idempotent)
- Process a full or partial refund against a completed payment
- Query payment status
- Record every attempt/transition to an immutable audit log
- Handle a provider being down — fail over to a backup provider for the same method, or fail fast

---

## 4. Classes

|Class|Responsibility|
|---|---|
|**PaymentGateway**|Top-level facade — the single entry point clients call to `charge()`, `refund()`, `getStatus()` — see [[facade]]|
|**Payment**|Central entity: amount, currency, method, current [[state]], idempotency key, provider reference|
|**PaymentState (interface)**|Declares state-dependent operations (`markSuccess()`, `markFailed()`, `refund()`)|
|**PendingState / SuccessState / FailedState / RefundedState** (concrete states)|Each governs valid transitions for that stage|
|**PaymentMethodStrategy (interface)**|Declares `process(payment)` — concrete methods (`CardPayment`, `WalletPayment`, `UPIPayment`) implement this — see [[strategy]]|
|**ProviderAdapter (interface)**|Wraps a specific external provider's API in a common interface — see Adapter reasoning below|
|**ProviderRouter**|Selects which `ProviderAdapter` to use for a given method (supports failover to a backup provider)|
|**PaymentProcessor (proxy)**|The client-facing gateway's local stand-in for the remote provider call — a [[proxy]]|
|**IdempotencyStore**|Tracks idempotency keys and their associated result, so retried requests return the original result instead of reprocessing — directly implements [[idempotency]]'s key pattern|
|**AuditLogger**|Immutable log of every payment attempt/transition|
|**PaymentObserver (interface)**|Declares `update(payment)` — implemented by merchant-notification/webhook-dispatch components — see [[observer]]|
|**WebhookListener**|Receives async provider callbacks, updates the corresponding `Payment`'s state|
|**RefundProcessor**|Coordinates a refund against a previously successful `Payment`, delegating to the same `ProviderAdapter` used for the original charge|

---

## 5. Relationships

- **PaymentGateway "has-a" ProviderRouter, IdempotencyStore, AuditLogger** — composition; the facade coordinates all of them for a single `charge()` call
- **Payment "has-a" PaymentState** — composition; delegates lifecycle behavior to its current state (State pattern, same structural approach as every prior LLD note in this vault)
- **PaymentGateway "uses-a" PaymentMethodStrategy** — Strategy pattern; the specific payment method's processing logic is interchangeable behind a common interface
- **ProviderRouter "uses-many" ProviderAdapter** — each adapter wraps one external provider's specific API/SDK behind the same interface, so the router (and everything above it) never depends on a specific provider's quirks — this is fundamentally an **Adapter** pattern application (translating each provider's distinct interface into one common shape), used alongside [[proxy]] for the remote-call abstraction itself
- **Payment "is observed by" merchant notification / WebhookListener-driven updates** — Observer pattern; state changes trigger downstream notifications without Payment needing to know who's listening
- **IdempotencyStore "guards" PaymentGateway.charge()** — every charge attempt is first checked against the store; a duplicate key returns the previously stored result rather than reprocessing (see [[idempotency]])
- **RefundProcessor "references" Payment and ProviderAdapter** — a refund is only valid against a `Payment` in `SuccessState`, and is routed back through the same adapter/provider that processed the original charge

---

## 6. Design Patterns

- **[[state]]** — a `Payment`'s lifecycle (`Pending → Success/Failed`, with `Success → Refunded` as a further transition) is modeled as a state machine, the same structural reasoning as [[atm]], [[food-delivery]], [[movie-booking]], and others. This makes it structurally impossible to, say, refund a `Failed` or still-`Pending` payment.
- **[[strategy]]** — `PaymentMethodStrategy` lets the system support Card, Wallet, UPI, and BNPL as interchangeable implementations of a common `process(payment)` operation, so adding a new method doesn't require touching the `PaymentGateway`'s orchestration logic.
- **Adapter** (closely related to, and often confused with, [[proxy]] in interviews) — each `ProviderAdapter` translates one external provider's specific API shape (different field names, different response formats, different error codes) into the system's own common `ProviderAdapter` interface. This is Adapter's classic use case: making incompatible interfaces compatible, as opposed to Proxy's job of controlling access to an object that already shares the client's expected interface.
- **[[proxy]]** — beyond the adapter translation, the `PaymentProcessor` acts as a Remote Proxy for the actual network call to whichever provider was selected — hiding retries, timeouts, and serialization behind what looks like a simple local method call to the rest of the system.
- **[[observer]]** — `Payment` notifies registered observers (merchant webhook dispatch, internal analytics, notification service) on every state transition, decoupling "what happened to this payment" from "who needs to react."
- **[[facade]]** — `PaymentGateway` is the single, simple entry point (`charge()`, `refund()`, `getStatus()`) that internally coordinates idempotency checking, method-strategy selection, provider routing, state transitions, and audit logging — client code never touches those subsystems directly.
- **[[factory-method]]** — a `PaymentMethodFactory` creates the correct `PaymentMethodStrategy` implementation based on the requested method type, keeping that decision out of the `PaymentGateway`'s core `charge()` logic.
- **[[singleton]]** — `IdempotencyStore` and `AuditLogger` are natural single-instance-per-system components; in production these are best backed by a shared, durable store (e.g. [[redis]] or a database table) rather than an in-process singleton, since the idempotency guarantee must hold across multiple gateway server instances (see [[singleton]] trade-offs).

---

## 7. Class Diagram

```text
                    ┌────────────────────────┐
                    │      PaymentGateway        │ (Facade)
                    │ - idempotencyStore            │
                    │ - auditLogger                    │
                    │ - providerRouter                     │
                    │ + charge(request, idemKey): Payment    │
                    │ + refund(paymentId, amount)                │
                    │ + getStatus(paymentId)                       │
                    └─────────┬──────────────────────┘
                              │ coordinates
     ┌───────────────────────┼────────────────────────┐
     ▼                        ▼                          ▼
┌────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│ IdempotencyStore   │  │  ProviderRouter         │  │    AuditLogger           │
│ + checkOrReserve(key)│  │ + route(method): Adapter  │  └──────────────────────┘
│ + storeResult(key,r) │  └──────────┬───────────┘
└────────────────┘             │ selects among
                                 ▼
                       ┌──────────────────────┐
                       │ ProviderAdapter (interface)│
                       │ + charge(payment)            │
                       │ + refund(payment, amount)      │
                       └─────────┬─────────────┘
                    ┌────────────┼────────────┐
                    ▼              ▼             ▼
            ┌──────────────┐┌──────────────┐┌──────────────┐
            │ StripeAdapter  ││ RazorpayAdapter││ PayPalAdapter │
            └──────────────┘└──────────────┘└──────────────┘

┌────────────────┐        ┌──────────────────────┐
│      Payment       │───────▶│  PaymentState (interface)│
│ - amount, currency    │        │ + markSuccess()             │
│ - method                │        │ + markFailed()                │
│ - idempotencyKey          │        │ + refund(amount)                │
│ - observers: List            │        └─────────┬─────────────┘
│ + notifyObservers()            │                  │ implemented by
└──────────┬─────────────┘          ┌────────────────┼───────────────┐
           │ notifies                 ▼                  ▼                 ▼
 ┌──────────────────┐       ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
 │ PaymentObserver (if)│       │ PendingState   │  │SuccessState    │  │FailedState     │
 │  + update(payment)    │       └──────────────┘  └──────┬───────┘  └──────────────┘
 └─────────┬─────────┘                                     │
           │ implemented by                                  ▼
           ▼                                        ┌──────────────┐
 ┌──────────────────┐                               │ RefundedState  │
 │ WebhookListener /   │                             └──────────────┘
 │ MerchantNotifier     │
 └──────────────────┘

┌────────────────────┐
│ PaymentMethodStrategy(if)│
│ + process(payment)         │
└─────────┬─────────────┘
   ┌───────┼────────┐
   ▼         ▼          ▼
┌─────┐┌──────────┐┌─────┐
│Card ││ Wallet     ││ UPI  │
└─────┘└──────────┘└─────┘
```

---

## 8. Sequence Flow

```text
Charge — Happy Path with Idempotency Guard

Client    PaymentGateway   IdempotencyStore   ProviderRouter   ProviderAdapter   Payment(state)   AuditLogger
   │ charge(req, key=K1) │                    │                  │                 │                │
   │─────────────────────▶│                    │                  │                 │                │
   │                       │──checkOrReserve(K1)▶│                  │                 │                │
   │                       │◀── not seen before ─│ (reserved, marks │                 │                │
   │                       │                    │  in-progress)     │                 │                │
   │                       │ (create Payment, PendingState)          │                 │                │
   │                       │──log(K1, "charge attempt")───────────────┼─────────────────┼───────────────▶│
   │                       │──route(method)──────────────────────────▶│                 │                │
   │                       │◀──── StripeAdapter ──────────────────────│                 │                │
   │                       │──charge(payment)─────────────────────────┼────────────────▶│                │
   │                       │◀──── success ─────────────────────────────┼─────────────────│                │
   │                       │──markSuccess()───────────────────────────┼─────────────────▶│(SuccessState)  │
   │                       │──storeResult(K1, SUCCESS)──▶│              │                 │                │
   │                       │──notifyObservers()─────────┼──────────────┼─────────────────┼───────────────▶│
   │◀── Payment(SUCCESS) ─│                    │                  │                 │                │

Retry with the SAME idempotency key (e.g. client timeout, retries per [[retry-and-timeout]]):

Client    PaymentGateway   IdempotencyStore
   │ charge(req, key=K1)  │                    │
   │─────────────────────▶│                    │
   │                       │──checkOrReserve(K1)▶│
   │                       │◀── already SUCCESS ─│ (returns the ORIGINAL stored
   │                       │                    │  result — no new charge attempted)
   │◀── Payment(SUCCESS), same result ──────────│

Refund — against a SuccessState payment

Client    PaymentGateway   Payment(state)   ProviderAdapter   AuditLogger
   │ refund(paymentId, $20)│                │                 │
   │───────────────────────▶│                │                 │
   │                        │──refund($20)───▶│(SuccessState    │
   │                        │                │ .refund():       │
   │                        │                │ delegates to same│
   │                        │                │ provider adapter │
   │                        │                │ used originally,  │
   │                        │                │ -> RefundedState) │
   │                        │──refund(payment,$20)──────────────▶│
   │                        │◀──── success ──────────────────────│
   │                        │──log(refund)───────────────────────┼───────────────▶│
   │◀── Payment(REFUNDED) ─│                │                 │

Failure branch: if the provider adapter times out mid-charge (unknown outcome —
not a clean success or failure response), the Payment must remain in a distinct
"Pending/Uncertain" sub-state rather than being marked Failed outright, and a
reconciliation job should query the provider's own status endpoint later to
resolve it — marking it Failed prematurely risks a client incorrectly retrying
a charge that actually succeeded on the provider's side.
```

---

## 9. Code

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <memory>
#include <optional>

// ---------- Payment method / provider abstractions ----------
struct ChargeRequest {
    double amount;
    std::string currency;
    std::string method; // "CARD", "WALLET", "UPI"
};

class Payment; // fwd decl

class ProviderAdapter {
public:
    virtual ~ProviderAdapter() = default;
    virtual bool charge(double amount) = 0;
    virtual bool refund(double amount) = 0;
    virtual std::string providerName() const = 0;
};

// Adapter — translates a specific (imagined) external provider SDK into our common interface
class StripeAdapter : public ProviderAdapter {
public:
    bool charge(double amount) override {
        std::cout << "[Stripe] Charging $" << amount << "\n";
        return true; // simplified: assume success
    }
    bool refund(double amount) override {
        std::cout << "[Stripe] Refunding $" << amount << "\n";
        return true;
    }
    std::string providerName() const override { return "Stripe"; }
};

class RazorpayAdapter : public ProviderAdapter {
public:
    bool charge(double amount) override {
        std::cout << "[Razorpay] Charging $" << amount << "\n";
        return true;
    }
    bool refund(double amount) override {
        std::cout << "[Razorpay] Refunding $" << amount << "\n";
        return true;
    }
    std::string providerName() const override { return "Razorpay"; }
};

// ---------- Provider Router ----------
class ProviderRouter {
public:
    ProviderAdapter* route(const std::string& method) {
        // simplified static routing; real system would consult config/health checks
        static StripeAdapter stripe;
        static RazorpayAdapter razorpay;
        if (method == "UPI") return &razorpay;
        return &stripe;
    }
};

// ---------- Audit Logger ----------
class AuditLogger {
public:
    void log(const std::string& idemKey, const std::string& event) {
        std::cout << "[AUDIT] key=" << idemKey << " event=" << event << "\n";
    }
};

// ---------- Idempotency Store ----------
enum class IdemResult { NOT_FOUND, IN_PROGRESS, SUCCESS, FAILED };

class IdempotencyStore {
    std::unordered_map<std::string, IdemResult> store;
public:
    IdemResult check(const std::string& key) {
        auto it = store.find(key);
        return it == store.end() ? IdemResult::NOT_FOUND : it->second;
    }
    void reserve(const std::string& key) { store[key] = IdemResult::IN_PROGRESS; }
    void storeResult(const std::string& key, IdemResult result) { store[key] = result; }
};

// ---------- Payment States ----------
class PaymentState {
public:
    virtual ~PaymentState() = default;
    virtual void markSuccess(Payment& payment) { std::cout << "Cannot mark success in current state\n"; }
    virtual void markFailed(Payment& payment) { std::cout << "Cannot mark failed in current state\n"; }
    virtual bool refund(Payment& payment, double amount) { std::cout << "Cannot refund in current state\n"; return false; }
    virtual std::string name() const = 0;
};

// ---------- Payment (Context + Subject) ----------
class Payment {
    double amount;
    std::string idemKey;
    ProviderAdapter* adapter;
    std::unique_ptr<PaymentState> state;

public:
    Payment(double amt, std::string key, ProviderAdapter* ad)
        : amount(amt), idemKey(std::move(key)), adapter(ad);
};
```

> **Note:** the fragment above is intentionally incomplete for readability of the state-wiring order; the full, compilable version follows as a single consolidated listing (as with the [[library]] note's C++ section):

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <memory>

struct ChargeRequest {
    double amount;
    std::string currency;
    std::string method;
};

class ProviderAdapter {
public:
    virtual ~ProviderAdapter() = default;
    virtual bool charge(double amount) = 0;
    virtual bool refund(double amount) = 0;
};

class StripeAdapter : public ProviderAdapter {
public:
    bool charge(double amount) override { std::cout << "[Stripe] Charging $" << amount << "\n"; return true; }
    bool refund(double amount) override { std::cout << "[Stripe] Refunding $" << amount << "\n"; return true; }
};

class RazorpayAdapter : public ProviderAdapter {
public:
    bool charge(double amount) override { std::cout << "[Razorpay] Charging $" << amount << "\n"; return true; }
    bool refund(double amount) override { std::cout << "[Razorpay] Refunding $" << amount << "\n"; return true; }
};

class ProviderRouter {
public:
    ProviderAdapter* route(const std::string& method) {
        static StripeAdapter stripe;
        static RazorpayAdapter razorpay;
        return method == "UPI" ? static_cast<ProviderAdapter*>(&razorpay) : static_cast<ProviderAdapter*>(&stripe);
    }
};

class AuditLogger {
public:
    void log(const std::string& idemKey, const std::string& event) {
        std::cout << "[AUDIT] key=" << idemKey << " event=" << event << "\n";
    }
};

enum class IdemResult { NOT_FOUND, IN_PROGRESS, SUCCESS, FAILED };

class IdempotencyStore {
    std::unordered_map<std::string, IdemResult> store;
public:
    IdemResult check(const std::string& key) {
        auto it = store.find(key);
        return it == store.end() ? IdemResult::NOT_FOUND : it->second;
    }
    void reserve(const std::string& key) { store[key] = IdemResult::IN_PROGRESS; }
    void storeResult(const std::string& key, IdemResult result) { store[key] = result; }
};

class Payment;

class PaymentState {
public:
    virtual ~PaymentState() = default;
    virtual void markSuccess(Payment& p) {}
    virtual void markFailed(Payment& p) {}
    virtual bool refund(Payment& p, double amount) { std::cout << "Cannot refund in current state\n"; return false; }
    virtual std::string name() const = 0;
};

class Payment {
    double amount;
    std::string idemKey;
    ProviderAdapter* adapter;
    std::unique_ptr<PaymentState> state;

public:
    Payment(double amt, std::string key, ProviderAdapter* ad);

    double getAmount() const { return amount; }
    ProviderAdapter* getAdapter() const { return adapter; }
    void setState(std::unique_ptr<PaymentState> s) { state = std::move(s); }
    std::string currentStateName() const { return state->name(); }

    void markSuccess() { state->markSuccess(*this); }
    void markFailed() { state->markFailed(*this); }
    bool refund(double amt) { return state->refund(*this, amt); }
};

class FailedState : public PaymentState {
public:
    std::string name() const override { return "Failed"; }
};

class RefundedState : public PaymentState {
public:
    std::string name() const override { return "Refunded"; }
};

class SuccessState : public PaymentState {
public:
    bool refund(Payment& p, double amount) override {
        if (p.getAdapter()->refund(amount)) {
            p.setState(std::make_unique<RefundedState>());
            return true;
        }
        return false;
    }
    std::string name() const override { return "Success"; }
};

class PendingState : public PaymentState {
public:
    void markSuccess(Payment& p) override { p.setState(std::make_unique<SuccessState>()); }
    void markFailed(Payment& p) override { p.setState(std::make_unique<FailedState>()); }
    std::string name() const override { return "Pending"; }
};

Payment::Payment(double amt, std::string key, ProviderAdapter* ad)
    : amount(amt), idemKey(std::move(key)), adapter(ad) {
    state = std::make_unique<PendingState>();
}

// ---------- PaymentGateway (Facade) ----------
class PaymentGateway {
    IdempotencyStore idemStore;
    ProviderRouter router;
    AuditLogger logger;
    std::unordered_map<std::string, std::unique_ptr<Payment>> payments; // keyed by idemKey for this demo

public:
    Payment* charge(const ChargeRequest& req, const std::string& idemKey) {
        IdemResult existing = idemStore.check(idemKey);
        if (existing == IdemResult::SUCCESS || existing == IdemResult::FAILED) {
            std::cout << "Idempotent replay — returning original result for key " << idemKey << "\n";
            return payments[idemKey].get();
        }
        idemStore.reserve(idemKey);
        logger.log(idemKey, "charge attempt");

        ProviderAdapter* adapter = router.route(req.method);
        auto payment = std::make_unique<Payment>(req.amount, idemKey, adapter);
        Payment* ptr = payment.get();
        payments[idemKey] = std::move(payment);

        if (adapter->charge(req.amount)) {
            ptr->markSuccess();
            idemStore.storeResult(idemKey, IdemResult::SUCCESS);
            logger.log(idemKey, "charge success");
        } else {
            ptr->markFailed();
            idemStore.storeResult(idemKey, IdemResult::FAILED);
            logger.log(idemKey, "charge failed");
        }
        return ptr;
    }

    bool refund(const std::string& idemKey, double amount) {
        auto it = payments.find(idemKey);
        if (it == payments.end()) { std::cout << "Payment not found\n"; return false; }
        bool ok = it->second->refund(amount);
        logger.log(idemKey, ok ? "refund success" : "refund rejected");
        return ok;
    }
};

// ---------- Demo ----------
int main() {
    PaymentGateway gateway;

    ChargeRequest req{49.99, "USD", "CARD"};
    auto* payment = gateway.charge(req, "idem-key-001");
    std::cout << "Status: " << payment->currentStateName() << "\n";

    // Client retries with the SAME idempotency key (e.g. after a timeout) — no double charge
    auto* replay = gateway.charge(req, "idem-key-001");
    std::cout << "Replay status: " << replay->currentStateName() << "\n";

    gateway.refund("idem-key-001", 49.99);
    std::cout << "Status after refund: " << payment->currentStateName() << "\n";

    return 0;
}
```

---

## 10. Trade-offs

- **In-memory IdempotencyStore vs a durable, shared store** — the demo uses an in-process map; a real, horizontally-scaled payment gateway must back this with a durable, shared store (e.g. [[redis]] with `SET NX` + TTL, or a database table with a unique constraint on the idempotency key) so the guarantee holds across multiple gateway instances and survives restarts — an in-memory-only store would silently lose the double-charge protection on failover.
- **Synchronous provider response vs asynchronous webhook confirmation** — some payment methods (bank transfers, certain BNPL flows) don't confirm synchronously; the design must support a `Payment` sitting in `PendingState` for an extended period until a `WebhookListener` later calls `markSuccess()`/`markFailed()` — this is more complex than the demo's fully-synchronous flow but is unavoidable for those methods.
- **Marking Failed on timeout vs a distinct "Uncertain" state** — as called out in the sequence flow, a provider timeout is genuinely ambiguous (the charge may have succeeded on their end even though the response was lost) — treating it identically to a definite failure risks an incorrect retry causing a real double-charge; a more robust design adds an explicit intermediate state requiring reconciliation against the provider's own status API before resolving to Success/Failed.
- **Static provider routing vs health-aware failover** — the demo's `ProviderRouter` always routes a method to the same adapter; a production system should track each provider's recent success/error rate (a natural fit for the [[retry-and-timeout]] circuit breaker pattern) and fail over to a backup provider for that method if the primary is degraded, trading a bit of routing complexity for meaningfully better availability.
- **Refund always through the original provider adapter** — this design (correctly) routes a refund back through whichever adapter processed the original charge, since a provider can generally only refund a charge it itself processed; this constrains any future provider-switching logic to apply only to _new_ charges, not retroactively to old ones.

---

## 11. Interview Questions

- **How do you guarantee a retried charge request never results in a double charge?** This is the idempotency key pattern (see [[idempotency]]): before attempting any charge, check a durable store keyed by the client-supplied idempotency key; if a result already exists, return it directly without re-attempting the charge. The check-and-reserve step must itself be atomic (e.g. a database unique constraint or `SET NX` in Redis) to prevent a race where two near-simultaneous retries both pass the "not found" check before either records a result.
- **What happens if the payment provider times out — you don't know if the charge succeeded or failed?** This is the hardest real case in payment systems: never mark it `Failed` outright, since a client-side retry could then cause a genuine double charge if it actually succeeded upstream. Instead, keep it in a distinct pending/uncertain state and reconcile by querying the provider's own status/lookup API asynchronously (a background job), only resolving the state once a definitive answer is obtained — see [[retry-and-timeout]] for the broader pattern of handling ambiguous failures.
- **Why model the provider integrations as Adapters rather than each having its own bespoke calling code scattered through the system?** Different providers have different APIs, field names, error formats, and authentication schemes; the Adapter pattern normalizes all of that behind one common `ProviderAdapter` interface, so the rest of the system (`ProviderRouter`, `PaymentGateway`) is written once against that common shape and doesn't need to change when a new provider is added or an existing one's API changes — only that provider's adapter implementation needs updating.
- **How would you support automatic failover if the primary provider for a payment method is down?** Extend `ProviderRouter` to track each adapter's recent health (error rate, response time — conceptually a [[retry-and-timeout]] circuit breaker per provider) and route to a healthy backup adapter for that method when the primary's circuit is open, rather than failing every request outright while the primary recovers.
- **How would you ensure sensitive card data is never exposed to your own system's logs or database?** Use tokenization at the point of entry — the client-facing layer (often the provider's own hosted fields/SDK) exchanges raw card details directly with the provider for a token, and your system only ever stores and passes around that token, never the underlying PAN/CVV — this keeps the payment system itself largely out of PCI-DSS scope for raw cardholder data.
- **Why use the State pattern for Payment instead of a simple status enum with validation in the API layer?** As with every other LLD problem in this vault, centralizing valid transitions in state-specific classes (e.g. `refund()` is only meaningful from `SuccessState`) makes invalid operations (refunding a `Pending` or `Failed` payment) structurally rejected at the point of the state object itself, rather than relying on every caller remembering to check the status field correctly before acting.

---

## Related Notes

- [[atm]]
- [[elevator]]
- [[food-delivery]]
- [[library]]
- [[movie-booking]]
- [[parking-lot]]
- [[state]]
- [[strategy]]
- [[observer]]
- [[facade]]
- [[proxy]]
- [[factory-method]]
- [[singleton]]
- [[idempotency]]
- [[retry-and-timeout]]
- [[reliability]]
- [[redis]]