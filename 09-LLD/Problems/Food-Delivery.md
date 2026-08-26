# Food Delivery System — Low Level Design

## 1. Requirements

### Functional

- Customer can browse restaurants (by location/cuisine/search), view menus, and place an order
- Customer can track order status in real time (placed → accepted → preparing → picked up → delivered)
- Restaurant can accept/reject an incoming order, update menu/item availability, mark order "ready for pickup"
- System assigns a delivery agent to an accepted order (matching)
- Delivery agent can accept/reject an assigned delivery, navigate to restaurant then customer, mark pickup/delivery complete
- Customer can make a payment (multiple methods); system handles payment capture and, if needed, refunds
- Customer can rate/review the restaurant and delivery agent after delivery
- Customer can cancel an order within an allowed window; system handles partial/full refund logic accordingly
- Notifications sent to customer, restaurant, and delivery agent at each order state transition

### Non-Functional

- **Consistency** — an order's state must be unambiguous across customer app, restaurant app, and delivery agent app at all times — no two parties should see conflicting statuses (ties to [[consistency]])
- **Availability** — placing an order and tracking it must remain highly available even during peak hours (ties to [[availability]])
- **Scalability** — must handle large spikes (lunch/dinner rush) — both order volume and real-time location updates from thousands of delivery agents (ties to [[scalability]], [[horizontal-vs-vertical-scaling]])
- **Low latency matching** — assigning a delivery agent to an order should happen within seconds, not minutes (ties to [[latency]])
- **Reliability** — a payment must never be captured without a corresponding valid order, and vice versa — no lost orders, no double charges (ties to [[reliability]], [[idempotency]])
- **Real-time updates** — location tracking and status changes should propagate to the customer's app with minimal delay (natural fit for [[websocket]] or push notifications)
- **Extensibility** — new order types (scheduled orders, group orders, subscription meals) should be addable without restructuring the core order/state model

---

## 2. Actors

- **Customer** — browses, orders, pays, tracks, rates
- **Restaurant** — manages menu, accepts/prepares orders
- **Delivery Agent** — accepts deliveries, picks up, delivers
- **Payment Gateway** (external system) — processes payments/refunds
- **Notification Service** (system-internal actor) — pushes status updates to all parties
- **Matching/Dispatch Service** (system-internal actor) — assigns delivery agents to orders

---

## 3. Use Cases

- Search restaurants and view menu
- Add items to cart, place order
- Restaurant accepts or rejects an order
- System matches and assigns a delivery agent
- Delivery agent accepts assignment, navigates, picks up, delivers
- Customer tracks live order status/location
- Process payment at order placement; process refund on cancellation
- Cancel an order (with time-window-dependent refund logic)
- Rate restaurant and delivery agent post-delivery
- Handle no delivery agent available / restaurant rejects order (fallback flows)

---

## 4. Classes

|Class|Responsibility|
|---|---|
|**FoodDeliverySystem**|Top-level facade coordinating order placement, matching, and payment — see [[facade]]|
|**Customer / Restaurant / DeliveryAgent**|Core actor entities holding profile, location, and relevant references|
|**MenuItem / Menu**|Restaurant's catalog of orderable items and prices|
|**Order**|Central entity; holds items, customer, restaurant, delivery agent, current [[state]], and total amount|
|**OrderState (interface)**|Declares state-dependent operations (`accept()`, `reject()`, `markPickedUp()`, `markDelivered()`, `cancel()`)|
|**PlacedState / AcceptedState / PreparingState / ReadyForPickupState / OutForDeliveryState / DeliveredState / CancelledState** (concrete states)|Each governs valid transitions for that lifecycle stage|
|**DispatchService**|Matches an accepted order to a delivery agent using a pluggable [[strategy]]|
|**MatchingStrategy (interface)**|Declares `findAgent(order, availableAgents)` — different assignment algorithms implement this|
|**NearestAgentStrategy / LoadBalancedStrategy** (concrete strategies)|Concrete delivery-agent matching algorithms|
|**PaymentProcessor (interface / proxy)**|Gateway to the external payment provider — a [[proxy]], since the real processor lives remotely|
|**PaymentStrategy (interface)**|Declares `pay(amount)` — concrete payment methods (Card, Wallet, UPI, CashOnDelivery) implement this|
|**OrderObserver (interface)**|Declares `update(order)` — implemented by Notification, tracking Display components — see [[observer]]|
|**NotificationService**|Observes Order state changes, pushes notifications to customer/restaurant/agent|
|**OrderFactory**|Creates the correct `Order` subtype (`StandardOrder`, `ScheduledOrder`) via [[factory-method]]|
|**Rating / Review**|Post-delivery feedback tied to an order, restaurant, and delivery agent|

---

## 5. Relationships

- **FoodDeliverySystem "has-a" DispatchService, PaymentProcessor, NotificationService** — composition; the facade coordinates all of them for a single `placeOrder()` call
- **Order "has-a" OrderState** — composition; Order delegates all lifecycle behavior to its current state (State pattern, same structural approach as the [[atm|ATM]] and [[elevator]] designs)
- **DispatchService "uses-a" MatchingStrategy** — Strategy pattern; the dispatch algorithm is swappable independent of the DispatchService's own coordination logic
- **PaymentProcessor "uses-a" PaymentStrategy** — Strategy pattern; the specific payment method is interchangeable behind a common `pay()` interface
- **Order "is observed by" NotificationService (and any tracking display)** — Observer pattern; Order (Subject) notifies observers on every state transition without knowing what they do with that information
- **Order "is-a" abstract base**, with `StandardOrder` and `ScheduledOrder` as subclasses — inheritance; `OrderFactory` decides which to create based on whether the customer specified a future delivery time
- **Restaurant "has-a" Menu, has-many MenuItem** — composition
- **Order "references" Customer, Restaurant, DeliveryAgent** — association, resolved at order-creation and assignment time respectively (DeliveryAgent reference is set only after successful matching, not at order creation)

---

## 6. Design Patterns

- **[[state]]** — Order lifecycle (`Placed → Accepted → Preparing → ReadyForPickup → OutForDelivery → Delivered`, with a `Cancelled` branch reachable from most non-terminal states) is modeled as a state machine, exactly the same structural reasoning as [[atm]] and [[elevator]]. This makes invalid transitions (e.g. marking a `Placed` order as `Delivered` directly) structurally impossible rather than something enforced ad hoc with conditionals.
- **[[strategy]]** — used **twice** in this design: (1) `MatchingStrategy` for delivery-agent assignment (nearest agent vs load-balanced across agents vs a zone-based strategy for very large cities), and (2) `PaymentStrategy` for the payment method (Card, UPI, Wallet, Cash on Delivery) — both let the system swap algorithms/methods without touching the coordinating service's core logic.
- **[[observer]]** — the Order notifies the `NotificationService` (and potentially a live-tracking display) on every state change, decoupling "what happened to the order" from "who needs to know and what they do about it." New notification channels (SMS, push, email) can subscribe without modifying Order itself.
- **[[factory-method]]** — `OrderFactory` creates the correct `Order` subtype (immediate `StandardOrder` vs a future-dated `ScheduledOrder`) based on customer input, keeping that decision out of the checkout/controller code.
- **[[proxy]]** — `PaymentProcessor` is a Remote Proxy in front of the actual external payment gateway, so the rest of the system calls what looks like a simple local `charge()`/`refund()` interface while the Proxy hides network calls, retries, and serialization.
- **[[facade]]** — `FoodDeliverySystem` provides one simplified entry point (`placeOrder()`) that internally coordinates menu validation, payment, order creation, and dispatch — hiding that orchestration from the client (e.g. the API layer/controller) while still allowing direct access to subsystems (e.g. `DispatchService`) if a more advanced flow needs it.
- **[[command]]** (optional extension) — each customer action (place order, cancel order) could be modeled as a Command object, useful if the system needs an audit trail of every action taken, or wants to support "undo" for a cancellation-eligible window.

---

## 7. Class Diagram

```text
                    ┌────────────────────────┐
                    │  FoodDeliverySystem       │ (Facade)
                    │ - dispatchService           │
                    │ - paymentProcessor           │
                    │ - notificationService         │
                    │ + placeOrder(customer, items)  │
                    │ + cancelOrder(orderId)          │
                    └─────────┬──────────────┘
                              │ coordinates
        ┌─────────────────────┼─────────────────────┐
        ▼                      ▼                       ▼
┌────────────────┐  ┌──────────────────────┐ ┌──────────────────────┐
│ PaymentProcessor  │  │   DispatchService      │ │  NotificationService   │
│  (Proxy)           │  │ - strategy: MatchingStrategy│ │  (implements OrderObserver)│
│ + charge(amount)     │  │ + assignAgent(order)     │ └──────────────────────┘
│ + refund(amount)      │  └───────────┬──────────┘
└────────┬───────┘              │ uses
         │ uses                  ▼
         ▼              ┌──────────────────────────┐
┌────────────────┐    │ MatchingStrategy (interface) │
│PaymentStrategy(if)│    │ + findAgent(order, agents)    │
│ + pay(amount)      │    └─────────┬─────────────┘
└────────┬───────┘          ┌───────┴────────┐
   ┌─────┼──────┐            ▼                  ▼
   ▼      ▼        ▼  ┌──────────────┐  ┌──────────────────┐
┌─────┐┌─────┐┌─────┐ │NearestAgent   │  │LoadBalancedStrategy│
│Card ││UPI  ││Wallet│ │  Strategy      │  └──────────────────┘
└─────┘└─────┘└─────┘ └──────────────┘

┌────────────────────┐        ┌──────────────────────┐
│        Order         │───────▶│ OrderState (interface)  │
│ - id                   │        │ + accept()                │
│ - customer, restaurant  │        │ + reject()                 │
│ - deliveryAgent          │        │ + markReady()               │
│ - items, totalAmount      │        │ + markPickedUp()             │
│ - state                    │        │ + markDelivered()             │
│ - observers: List            │        │ + cancel()                     │
│ + notifyObservers()            │        └─────────┬─────────────┘
└──────────┬─────────────┘                  │ implemented by
           │ notifies              ┌─────────┼────────┬───────────┬──────────────┬─────────────┐
           ▼                        ▼           ▼          ▼            ▼               ▼             ▼
 ┌──────────────────┐     ┌──────────┐┌──────────────┐┌──────────────┐┌──────────────────┐┌────────────┐┌─────────────┐
 │ OrderObserver (if)  │     │PlacedState││AcceptedState  ││PreparingState ││ReadyForPickupState││OutForDelivery││DeliveredState│
 │  + update(order)      │     └──────────┘└──────────────┘└──────────────┘└──────────────────┘│State        │└─────────────┘
 └─────────┬─────────┘                                                                          └────────────┘
           │ implemented by                                                    ┌───────────────┐
           ▼                                                                    │CancelledState  │
 ┌──────────────────┐                                                          └───────────────┘
 │NotificationService │
 └──────────────────┘

┌────────────────┐
│ OrderFactory      │
│ + create(type, ...)│──creates──▶ StandardOrder / ScheduledOrder (both "is-a" Order)
└────────────────┘
```

---

## 8. Sequence Flow

```text
Place Order — Happy Path

Customer   FoodDeliverySystem   PaymentProcessor   OrderFactory   Order(state)   DispatchService   MatchingStrategy   Agent   NotificationService
  │ placeOrder(items)   │                    │                │              │                  │                  │        │
  │─────────────────────▶│                    │                │              │                  │                  │        │
  │                       │──charge(amount)───▶│                │              │                  │                  │        │
  │                       │◀──── success ──────│                │              │                  │                  │        │
  │                       │──create(STANDARD)───┼───────────────▶│              │                  │                  │        │
  │                       │                    │                │ (PlacedState)│                  │                  │        │
  │                       │                    │                │──notifyObservers()───────────────┼──────────────────┼───────▶│
  │                       │                    │                │              │                  │                  │ (customer +
  │                       │                    │                │              │                  │                  │  restaurant
  │                       │                    │                │              │                  │                  │  notified: "Order Placed")
  │  ... restaurant accepts (external actor call) ...            │              │                  │                  │        │
  │                       │                    │                │──accept()────▶│ (AcceptedState)   │                  │        │
  │                       │                    │                │──notifyObservers()───────────────┼──────────────────┼───────▶│
  │                       │                    │                │              │──assignAgent(order)▶│                  │        │
  │                       │                    │                │              │                  │──findAgent(order, agents)──▶│
  │                       │                    │                │              │                  │◀──── Agent #4 ───│        │
  │                       │                    │                │              │◀── Agent #4 ──────│                  │        │
  │                       │                    │                │──setDeliveryAgent(Agent #4)───────┼──────────────────▶│        │
  │                       │                    │                │──notifyObservers()───────────────┼──────────────────┼───────▶│
  │                       │                    │                │              │                  │                  │ (agent notified:
  │                       │                    │                │              │                  │                  │  "New delivery assigned")
  │  ... restaurant marks ready, agent picks up, delivers ...    │              │                  │                  │        │
  │                       │                    │                │──markDelivered()──▶(DeliveredState)                 │        │
  │                       │                    │                │──notifyObservers()───────────────┼──────────────────┼───────▶│
  │◀── "Order delivered" notification ─────────────────────────────────────────────────────────────────────────────────────────│

Failure branch: if DispatchService.assignAgent() finds no available agent,
Order remains in AcceptedState with a retry/backoff loop (see [[retry-and-timeout]]);
if the restaurant instead rejects the order at PlacedState, a compensating
refund via PaymentProcessor.refund() must be issued before transitioning
to CancelledState — same debit/compensate reasoning as the ATM's withdrawal flow.
```

---

## 9. Code

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>
#include <limits>
#include <cmath>

// ---------- Supporting types ----------
struct Location {
    double lat, lng;
};

double distance(const Location& a, const Location& b) {
    return std::sqrt(std::pow(a.lat - b.lat, 2) + std::pow(a.lng - b.lng, 2));
}

struct MenuItem {
    std::string name;
    double price;
};

// ---------- Payment (Strategy + Proxy) ----------
class PaymentStrategy {
public:
    virtual ~PaymentStrategy() = default;
    virtual bool pay(double amount) = 0;
    virtual std::string name() const = 0;
};

class CardPayment : public PaymentStrategy {
public:
    bool pay(double amount) override {
        std::cout << "Charged $" << amount << " to card\n";
        return true;
    }
    std::string name() const override { return "Card"; }
};

class CashOnDeliveryPayment : public PaymentStrategy {
public:
    bool pay(double amount) override {
        std::cout << "Cash on delivery: $" << amount << " due at drop-off\n";
        return true; // "success" here just means order can proceed
    }
    std::string name() const override { return "CashOnDelivery"; }
};

// Remote Proxy standing in for the real external payment gateway
class PaymentProcessor {
    std::unique_ptr<PaymentStrategy> strategy;

public:
    explicit PaymentProcessor(std::unique_ptr<PaymentStrategy> strat) : strategy(std::move(strat)) {}

    bool charge(double amount) { return strategy->pay(amount); }

    bool refund(double amount) {
        std::cout << "Refunded $" << amount << " via " << strategy->name() << "\n";
        return true;
    }
};

// ---------- Delivery Agent ----------
class DeliveryAgent {
    int id;
    Location location;
    bool available = true;

public:
    DeliveryAgent(int id_, Location loc) : id(id_), location(loc) {}
    int getId() const { return id; }
    Location getLocation() const { return location; }
    bool isAvailable() const { return available; }
    void setAvailable(bool a) { available = a; }
};

// ---------- Order forward declaration ----------
class Order;

// ---------- Observer ----------
class OrderObserver {
public:
    virtual ~OrderObserver() = default;
    virtual void update(const Order& order) = 0;
};

class NotificationService : public OrderObserver {
public:
    void update(const Order& order) override; // defined after Order
};

// ---------- Matching Strategy ----------
struct AssignmentRequest { Location restaurantLocation; };

class MatchingStrategy {
public:
    virtual ~MatchingStrategy() = default;
    virtual DeliveryAgent* findAgent(const AssignmentRequest& req, std::vector<std::unique_ptr<DeliveryAgent>>& agents) = 0;
};

class NearestAgentStrategy : public MatchingStrategy {
public:
    DeliveryAgent* findAgent(const AssignmentRequest& req, std::vector<std::unique_ptr<DeliveryAgent>>& agents) override {
        DeliveryAgent* best = nullptr;
        double bestDist = std::numeric_limits<double>::max();
        for (auto& a : agents) {
            if (!a->isAvailable()) continue;
            double d = distance(a->getLocation(), req.restaurantLocation);
            if (d < bestDist) { bestDist = d; best = a.get(); }
        }
        return best;
    }
};

// ---------- Order States ----------
class OrderState {
public:
    virtual ~OrderState() = default;
    virtual void accept(Order& order) { std::cout << "Cannot accept in current state\n"; }
    virtual void reject(Order& order) { std::cout << "Cannot reject in current state\n"; }
    virtual void markReady(Order& order) { std::cout << "Cannot mark ready in current state\n"; }
    virtual void markPickedUp(Order& order) { std::cout << "Cannot mark picked up in current state\n"; }
    virtual void markDelivered(Order& order) { std::cout << "Cannot mark delivered in current state\n"; }
    virtual void cancel(Order& order) { std::cout << "Cannot cancel in current state\n"; }
    virtual std::string name() const = 0;
};

// ---------- Order (Subject + State Context) ----------
class Order {
    int id;
    double totalAmount;
    std::vector<MenuItem> items;
    DeliveryAgent* deliveryAgent = nullptr;
    std::unique_ptr<OrderState> state;
    std::vector<OrderObserver*> observers;
    PaymentProcessor& payment;

public:
    Order(int id_, std::vector<MenuItem> its, PaymentProcessor& pay)
        : id(id_), items(std::move(its)), payment(pay) {
        totalAmount = 0;
        for (auto& i : items) totalAmount += i.price;
    }

    int getId() const { return id; }
    double getTotal() const { return totalAmount; }
    std::string currentStateName() const { return state->name(); }
    PaymentProcessor& paymentProcessor() { return payment; }

    void setState(std::unique_ptr<OrderState> newState) { state = std::move(newState); notifyObservers(); }
    void setDeliveryAgent(DeliveryAgent* agent) { deliveryAgent = agent; }
    DeliveryAgent* getDeliveryAgent() const { return deliveryAgent; }

    void addObserver(OrderObserver* o) { observers.push_back(o); }
    void notifyObservers() { for (auto* o : observers) o->update(*this); }

    // Delegate to current state
    void accept() { state->accept(*this); }
    void reject() { state->reject(*this); }
    void markReady() { state->markReady(*this); }
    void markPickedUp() { state->markPickedUp(*this); }
    void markDelivered() { state->markDelivered(*this); }
    void cancel() { state->cancel(*this); }
};

void NotificationService::update(const Order& order) {
    std::cout << "[Notify] Order " << order.getId() << " -> " << order.currentStateName() << "\n";
}

// ---------- Concrete States ----------
class CancelledState : public OrderState {
public:
    std::string name() const override { return "Cancelled"; }
};

class DeliveredState : public OrderState {
public:
    std::string name() const override { return "Delivered"; }
};

class OutForDeliveryState : public OrderState {
public:
    void markDelivered(Order& order) override {
        order.setState(std::make_unique<DeliveredState>());
    }
    std::string name() const override { return "OutForDelivery"; }
};

class ReadyForPickupState : public OrderState {
public:
    void markPickedUp(Order& order) override {
        order.setState(std::make_unique<OutForDeliveryState>());
    }
    std::string name() const override { return "ReadyForPickup"; }
};

class PreparingState : public OrderState {
public:
    void markReady(Order& order) override {
        order.setState(std::make_unique<ReadyForPickupState>());
    }
    void cancel(Order& order) override {
        order.paymentProcessor().refund(order.getTotal());
        order.setState(std::make_unique<CancelledState>());
    }
    std::string name() const override { return "Preparing"; }
};

class AcceptedState : public OrderState {
public:
    void markReady(Order& order) override { // simplified: skip straight to preparing->ready in demo
        order.setState(std::make_unique<PreparingState>());
    }
    void cancel(Order& order) override {
        order.paymentProcessor().refund(order.getTotal());
        order.setState(std::make_unique<CancelledState>());
    }
    std::string name() const override { return "Accepted"; }
};

class PlacedState : public OrderState {
public:
    void accept(Order& order) override {
        order.setState(std::make_unique<AcceptedState>());
    }
    void reject(Order& order) override {
        order.paymentProcessor().refund(order.getTotal()); // compensating refund
        order.setState(std::make_unique<CancelledState>());
    }
    void cancel(Order& order) override {
        order.paymentProcessor().refund(order.getTotal());
        order.setState(std::make_unique<CancelledState>());
    }
    std::string name() const override { return "Placed"; }
};

// ---------- Dispatch Service ----------
class DispatchService {
    std::unique_ptr<MatchingStrategy> strategy;
    std::vector<std::unique_ptr<DeliveryAgent>>& agents;

public:
    DispatchService(std::unique_ptr<MatchingStrategy> strat, std::vector<std::unique_ptr<DeliveryAgent>>& agentList)
        : strategy(std::move(strat)), agents(agentList) {}

    DeliveryAgent* assignAgent(const Location& restaurantLocation) {
        AssignmentRequest req{restaurantLocation};
        DeliveryAgent* chosen = strategy->findAgent(req, agents);
        if (chosen) chosen->setAvailable(false);
        return chosen;
    }
};

// ---------- FoodDeliverySystem (Facade) ----------
class FoodDeliverySystem {
    std::vector<std::unique_ptr<DeliveryAgent>> agents;
    DispatchService dispatch;
    NotificationService notifier;
    int nextOrderId = 1;

public:
    FoodDeliverySystem() : dispatch(std::make_unique<NearestAgentStrategy>(), agents) {
        agents.push_back(std::make_unique<DeliveryAgent>(1, Location{12.9, 77.6}));
        agents.push_back(std::make_unique<DeliveryAgent>(2, Location{12.95, 77.65}));
    }

    std::unique_ptr<Order> placeOrder(std::vector<MenuItem> items, PaymentProcessor& payment) {
        double total = 0;
        for (auto& i : items) total += i.price;
        if (!payment.charge(total)) {
            std::cout << "Payment failed, order not created\n";
            return nullptr;
        }
        auto order = std::make_unique<Order>(nextOrderId++, items, payment);
        order->addObserver(&notifier);
        order->setState(std::make_unique<PlacedState>());
        return order;
    }

    void onRestaurantAccept(Order& order, Location restaurantLocation) {
        order.accept();
        DeliveryAgent* agent = dispatch.assignAgent(restaurantLocation);
        if (agent) {
            order.setDeliveryAgent(agent);
            std::cout << "Assigned Agent " << agent->getId() << " to Order " << order.getId() << "\n";
        } else {
            std::cout << "No delivery agent available — will retry\n";
        }
    }
};

// ---------- Demo ----------
int main() {
    FoodDeliverySystem system;
    PaymentProcessor payment(std::make_unique<CardPayment>());

    auto order = system.placeOrder({{"Pizza", 12.99}, {"Coke", 2.50}}, payment);
    if (order) {
        system.onRestaurantAccept(*order, Location{12.91, 77.61});
        order->markReady();
        order->markPickedUp();
        order->markDelivered();
    }

    return 0;
}
```

---

## 10. Trade-offs

- **NearestAgentStrategy vs LoadBalancedStrategy** — always picking the nearest agent minimizes delivery time for one order but can concentrate assignments on agents near restaurant-dense areas, leaving others idle; a load-balanced or hybrid strategy trades a bit of per-order speed for better fleet-wide utilization — the same latency-vs-throughput tension seen in [[elevator]]'s scheduling discussion.
- **Compensating refund on cancel/reject vs a full distributed transaction** — this design issues a direct `refund()` call when an order is cancelled or rejected after payment was captured; a more robust production system would log the cancellation intent durably first (Saga-style, see [[acid]] section 6) so a crash between "order cancelled" and "refund issued" can be detected and reconciled, rather than silently losing the refund.
- **Central DispatchService vs sharded/zone-based dispatch** — a single DispatchService is simple to reason about but becomes a bottleneck at very high order volume across a large city; partitioning dispatch by geographic zone (similar to [[sharding]]'s reasoning, applied to a matching problem instead of data) trades some cross-zone optimality for horizontal scalability.
- **State pattern per order vs a generic status enum + validation function** — State pattern (used here, and in [[atm]]/[[elevator]]) makes invalid transitions structurally impossible and keeps each stage's logic isolated, at the cost of more classes; for a system with very few, rarely-changing order stages, a simpler enum + a transition-validation table might be sufficient.
- **Synchronous payment before order creation vs asynchronous payment confirmation** — charging synchronously (as shown) simplifies the flow but ties the customer's wait time to the payment gateway's latency; some real systems create the order in a "payment pending" state immediately and confirm asynchronously via a payment webhook, improving perceived responsiveness at the cost of a more complex initial state (ties to [[latency]], [[event-driven-architecture]]).

---

## 11. Interview Questions

- **How would you ensure a customer is never charged without a corresponding order being created, or vice versa?** Treat payment + order creation as a unit: attempt the charge first (reversible via refund), and only persist the order as `Placed` after the charge succeeds; if order persistence then fails, issue a compensating refund — the same debit/compensate pattern used in the [[atm]] withdrawal flow, and reinforced by using an [[idempotency|idempotency key]] tied to the checkout attempt so a client retry after a network blip doesn't double-charge.
- **How do you handle the case where no delivery agent is available near the restaurant?** Keep the order in `AcceptedState` and retry the `DispatchService.assignAgent()` call on a backoff schedule (see [[retry-and-timeout]]), notify the customer of the delay via the existing [[observer]] notification path, and potentially widen the search radius or offer a small incentive to agents slightly farther away if wait time exceeds a threshold.
- **How would you scale the DispatchService to handle a city with tens of thousands of concurrent orders during a lunch rush?** Partition dispatch by geographic zone so each partition only searches agents/orders within its area (analogous to [[sharding]]), and consider event-driven dispatch (agents' location updates and order-ready events flow through [[kafka]] or similar) rather than a single service synchronously querying all agents on every request.
- **Why use the State pattern for Order instead of a simple `status` string field with validation checks scattered in the API layer?** As the number of valid transitions grows (accept, reject, cancel — each valid only from specific prior states), scattering `if status == "placed"` checks across every endpoint handler becomes error-prone and easy to get inconsistent; State pattern centralizes each stage's valid operations into one class, making invalid transitions a compile-time-visible non-operation rather than a runtime bug waiting to happen — the same reasoning given in the [[atm]] and [[elevator]] designs.
- **How would you support scheduled/future-dated orders without duplicating the whole Order class?** Use [[factory-method]] — `OrderFactory` creates a `ScheduledOrder` subclass of the same abstract `Order` base when the customer specifies a future time, reusing the same `OrderState` machinery, but delaying the transition out of an initial "Scheduled" pre-state until the requested time arrives (a background job or timer triggers the transition into `Placed` at the right moment).
- **How would live location tracking be implemented so the customer sees the delivery agent moving in real time?** This is a natural fit for [[websocket]] (bidirectional, low-latency) rather than repeated polling — the delivery agent's app periodically publishes location updates (e.g. via [[pub-sub]] or a lightweight [[message-queues|message queue]]), and the customer's app subscribes to updates for their specific order, with the Order's [[observer]] mechanism extended to include a "location updated" event type alongside state-change events.

---

## Related Notes

- [[atm]]
- [[elevator]]
- [[state]]
- [[strategy]]
- [[observer]]
- [[factory-method]]
- [[facade]]
- [[proxy]]
- [[idempotency]]
- [[retry-and-timeout]]
- [[acid]]
- [[sharding]]
- [[websocket]]
- [[event-driven-architecture]]