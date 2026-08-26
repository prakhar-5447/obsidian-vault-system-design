# Movie Ticket Booking System — Low Level Design

## 1. Requirements

### Functional

- User can search movies (by title, city, language, genre) and view showtimes across theaters
- User can view a show's seat layout and select one or more seats
- User can hold selected seats temporarily while completing payment (prevents double-booking during checkout)
- User can complete payment and receive a confirmed booking (e-ticket)
- System releases held-but-unpaid seats automatically after a timeout
- User can cancel a booking (subject to a cancellation policy/time window)
- Admin/theater owner can add movies, screens, shows, and seat layouts
- System prevents two users from booking the same seat for the same show

### Non-Functional

- **Consistency** — a seat must never be sold to two different users for the same show, even under highly concurrent booking attempts (the defining hard requirement of this problem — ties to [[transactions]], [[isolation-levels]])
- **Availability** — search and browsing should remain available even under heavy load (e.g. a blockbuster's ticket window opening); booking itself may briefly queue but shouldn't be unavailable
- **Low latency** — seat-hold and payment confirmation should complete within a few seconds (ties to [[latency]])
- **Scalability** — must handle large simultaneous demand spikes for popular releases across many theaters/cities (ties to [[scalability]], [[horizontal-vs-vertical-scaling]])
- **Reliability** — a payment must never be captured without a corresponding confirmed seat, and vice versa (ties to [[reliability]], [[idempotency]])
- **Fairness** — under contention for the same seat, the system should resolve conflicts predictably (first successful hold wins), not silently favor one request

---

## 2. Actors

- **User (Customer)** — searches, selects seats, pays, books, cancels
- **Theater Admin** — manages movies, screens, shows, seat layouts, pricing
- **Payment Gateway** (external system) — processes payment/refunds
- **Notification Service** (system-internal actor) — sends booking confirmations, hold-expiry warnings

---

## 3. Use Cases

- Search movies/showtimes by city, date, language
- View seat layout for a specific show
- Hold one or more seats temporarily (checkout in progress)
- Complete payment and confirm booking
- Release a held seat automatically after hold-timeout expiry
- Cancel a confirmed booking (with refund policy applied)
- Admin adds a movie, screen, show, or seat layout
- Handle two users attempting to hold/book the same seat concurrently

---

## 4. Classes

|Class|Responsibility|
|---|---|
|**BookingSystem**|Top-level facade coordinating search, seat hold, payment, and booking confirmation — see [[facade]]|
|**Movie / Screen / Show**|Catalog entities: a Movie plays on a Screen at a specific Show (date/time)|
|**Seat**|One physical seat in a Screen's layout (row, number, type — Regular/Premium/Recliner)|
|**ShowSeat**|The booking-relevant entity: one Seat **for one specific Show**, holding its current [[state]] (Available, Held, Booked)|
|**ShowSeatState (interface)**|Declares state-dependent operations (`hold()`, `confirm()`, `release()`, `cancel()`)|
|**AvailableState / HeldState / BookedState** (concrete states)|Each governs valid transitions for that seat-for-a-show|
|**Booking**|A confirmed booking: user, show, list of ShowSeats, amount paid, booking time|
|**HoldTimer**|Tracks a hold's expiry; triggers automatic release if payment isn't completed in time|
|**PaymentProcessor (interface / proxy)**|Gateway to the external payment provider — a [[proxy]]|
|**PaymentStrategy (interface)**|Declares `pay(amount)` — concrete payment methods implement this — see [[strategy]]|
|**PricingStrategy (interface)**|Declares `calculatePrice(seat, show)` — pluggable pricing (flat, dynamic/surge, seat-type-based) — see [[strategy]]|
|**BookingObserver (interface)**|Declares `update(booking)` — implemented by NotificationService — see [[observer]]|
|**SeatLockManager**|Coordinates atomic hold-acquisition across concurrent requests for the same ShowSeat — the component that actually prevents double-booking|

---

## 5. Relationships

- **BookingSystem "has-a" SeatLockManager, PaymentProcessor, NotificationService** — composition; the facade coordinates all of them for a single `bookSeats()` flow
- **Show "has-many" ShowSeat** — one entry per physical Seat, scoped to that specific Show (the same physical Seat has an independent `ShowSeat` — and independent state — for every different Show it's part of)
- **ShowSeat "has-a" ShowSeatState** — composition; delegates behavior to its current state (State pattern, same structural approach as [[atm]], [[elevator]], [[food-delivery]], and [[library]])
- **BookingSystem "uses-a" PricingStrategy, uses-a PaymentStrategy** — Strategy pattern; both pricing and payment method are swappable independent of the booking flow itself
- **Booking "is observed by" NotificationService** — Observer pattern; booking confirmation/cancellation triggers notifications without the Booking needing to know who's listening
- **HoldTimer "monitors" ShowSeat** — association; on expiry, triggers the ShowSeat's `release()` transition back to `AvailableState` if payment wasn't completed in time
- **SeatLockManager "guards" ShowSeat.hold()** — the manager ensures that only one concurrent request can successfully transition a given ShowSeat out of `AvailableState`, typically backed by a distributed lock or an atomic conditional database update (see [[redis]]'s `SET NX` pattern, or a DB-level `UPDATE ... WHERE state = 'AVAILABLE'`)

---

## 6. Design Patterns

- **[[state]]** — each `ShowSeat`'s lifecycle (`Available → Held → Booked`, with `Held → Available` on timeout/cancellation-during-hold, and `Booked → Available` on a later cancellation) is modeled as a state machine — the same structural reasoning as every prior LLD note in this vault. This makes it structurally impossible to, say, confirm a seat that was never held, or hold a seat that's already booked.
- **[[strategy]]** — used **twice**: (1) `PricingStrategy` for ticket pricing (flat rate vs dynamic/surge pricing based on demand vs seat-type multipliers), and (2) `PaymentStrategy` for the payment method — both swappable without changing `BookingSystem`'s core flow.
- **[[observer]]** — `Booking` notifies `NotificationService` on confirmation/cancellation; the same mechanism can be extended so `HoldTimer` expiry notifications ("your seats were released, please re-select") flow through the same observer channel.
- **[[proxy]]** — `PaymentProcessor` is a Remote Proxy in front of the actual external payment gateway, hiding network calls/retries behind a simple `charge()`/`refund()` interface, as in [[food-delivery]].
- **[[facade]]** — `BookingSystem` exposes a small set of high-level operations (`searchShows()`, `holdSeats()`, `confirmBooking()`, `cancelBooking()`) that internally coordinate the SeatLockManager, PaymentProcessor, Booking creation, and notifications.
- **[[singleton]]** — `SeatLockManager` (and the underlying lock backend, e.g. [[redis]]) is a natural single, shared coordination point across the whole booking system — critical since seat-hold correctness depends on there being exactly one authority deciding who successfully acquires a hold.
- **[[factory-method]]** (optional extension) — a `BookingFactory` could construct different booking types if the system later supports group bookings, subscription passes, or corporate bulk bookings, each with slightly different confirmation rules, without changing the core booking flow.

---

## 7. Class Diagram

```text
                    ┌────────────────────────┐
                    │     BookingSystem          │ (Facade)
                    │ - seatLockManager             │
                    │ - paymentProcessor              │
                    │ - notificationService             │
                    │ - pricingStrategy                   │
                    │ + searchShows(query)                  │
                    │ + holdSeats(show, seats, user)          │
                    │ + confirmBooking(hold, payment)           │
                    │ + cancelBooking(bookingId)                  │
                    └─────────┬──────────────────────┘
                              │ coordinates
        ┌─────────────────────┼───────────────────────┐
        ▼                      ▼                         ▼
┌────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│ SeatLockManager    │  │  PaymentProcessor       │  │ NotificationService     │
│ + tryHold(showSeat)   │  │  (Proxy)                 │  │  (implements Observer)   │
│ + release(showSeat)    │  │ + charge(amount)           │  └──────────────────────┘
└────────────────┘  │ + refund(amount)             │
                       └──────────┬───────────┘
                                  │ uses
                                  ▼
                       ┌──────────────────────┐
                       │ PaymentStrategy (if)     │
                       │ + pay(amount)              │
                       └──────────┬───────────┘
                          ┌───────┴────────┐
                          ▼                  ▼
                  ┌──────────────┐  ┌──────────────┐
                  │  CardPayment   │  │  WalletPayment │
                  └──────────────┘  └──────────────┘

┌────────────────┐        ┌──────────────────────┐
│      Show          │───────▶│   ShowSeat              │
│ - movie, screen,      │        │ - seat (physical ref)    │
│   startTime             │        │ - state                    │
│ - seats: List<ShowSeat>  │        │ - observers: List             │
└────────────────┘        │ + notifyObservers()             │
                            └─────────┬─────────────┘
                                      │ delegates to
                                      ▼
                            ┌──────────────────────┐
                            │ShowSeatState (interface) │
                            │ + hold(user)                │
                            │ + confirm()                    │
                            │ + release()                       │
                            │ + cancel()                           │
                            └─────────┬─────────────┘
                          ┌────────────┼────────────┐
                          ▼              ▼             ▼
                  ┌──────────────┐┌──────────────┐┌──────────────┐
                  │AvailableState  ││ HeldState      ││ BookedState   │
                  └──────────────┘└──────────────┘└──────────────┘

┌────────────────┐        ┌──────────────────┐
│    Booking         │────────▶│   HoldTimer          │
│ - user, show, seats   │        │ + onExpire(showSeat)  │
│ - amountPaid            │        └──────────────────┘
│ - observers: List          │
└────────────────┘
```

---

## 8. Sequence Flow

```text
Book Seats — Happy Path (with concurrency guard)

User A       BookingSystem    SeatLockManager   ShowSeat(state)   PaymentProcessor   Booking    NotificationService
   │ holdSeats(show, [A1,A2])│                    │                 │                  │           │
   │─────────────────────────▶│                    │                 │                  │           │
   │                           │──tryHold(A1)───────▶│                 │                  │           │
   │                           │                    │──hold(userA)───▶│ (AvailableState  │           │
   │                           │                    │                 │  .hold(): success,│           │
   │                           │                    │                 │  -> HeldState)     │           │
   │                           │                    │◀──── success ──│                  │           │
   │                           │──tryHold(A2)───────▶│                 │                  │           │
   │                           │                    │──hold(userA)───▶│ (same as above)   │           │
   │                           │◀──── both held ─────────────────────│                  │           │
   │                           │ (start HoldTimer, e.g. 5 min)         │                  │           │
   │◀── seats held, proceed to payment ─────────────────────────────│                  │           │
   │ confirmBooking(payment)   │                    │                 │                  │           │
   │──────────────────────────▶│                    │                 │                  │           │
   │                           │──charge(amount)─────┼─────────────────┼──────────────────▶│           │
   │                           │◀──── success ────────────────────────┼──────────────────│           │
   │                           │──confirm()─────────▶│──confirm()─────▶│ (HeldState.confirm():│         │
   │                           │                    │                 │  -> BookedState)    │           │
   │                           │──createBooking()───┼─────────────────┼──────────────────▶│           │
   │                           │                    │                 │                  │──notifyObservers()──▶│
   │◀── booking confirmed, e-ticket ─────────────────────────────────────────────────────│           │

Concurrent conflict — User B tries the same seat while held by User A:

User B       BookingSystem    SeatLockManager   ShowSeat(state)
   │ holdSeats(show, [A1])    │                    │
   │─────────────────────────▶│                    │
   │                           │──tryHold(A1)───────▶│
   │                           │                    │──hold(userB)───▶ (HeldState.hold():
   │                           │                    │                  rejected — already
   │                           │                    │                  held by User A)
   │                           │◀──── failure ───────│
   │◀── "Seat A1 no longer available, please reselect" ─────────────│

Hold expiry (no payment completed in time):

HoldTimer                ShowSeat(state)
   │ onExpire()               │
   │─────────────────────────▶│──release()───▶ (HeldState.release():
   │                          │                 -> AvailableState,
   │                          │                 seat becomes bookable
   │                          │                 by anyone again)
```

---

## 9. Code

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>
#include <mutex>
#include <unordered_map>

// ---------- Payment (Strategy + Proxy) ----------
class PaymentStrategy {
public:
    virtual ~PaymentStrategy() = default;
    virtual bool pay(double amount) = 0;
};

class CardPayment : public PaymentStrategy {
public:
    bool pay(double amount) override {
        std::cout << "Charged $" << amount << " to card\n";
        return true;
    }
};

class PaymentProcessor {
    std::unique_ptr<PaymentStrategy> strategy;
public:
    explicit PaymentProcessor(std::unique_ptr<PaymentStrategy> s) : strategy(std::move(s)) {}
    bool charge(double amount) { return strategy->pay(amount); }
    void refund(double amount) { std::cout << "Refunded $" << amount << "\n"; }
};

// ---------- Notification (Observer) ----------
class Booking;

class BookingObserver {
public:
    virtual ~BookingObserver() = default;
    virtual void update(const Booking& booking) = 0;
};

class NotificationService : public BookingObserver {
public:
    void update(const Booking& booking) override; // defined after Booking
};

// ---------- ShowSeat States ----------
class ShowSeat; // fwd decl

class ShowSeatState {
public:
    virtual ~ShowSeatState() = default;
    virtual bool hold(ShowSeat& seat, const std::string& userId) { return false; }
    virtual bool confirm(ShowSeat& seat) { return false; }
    virtual void release(ShowSeat& seat) { std::cout << "Cannot release in current state\n"; }
    virtual std::string name() const = 0;
};

class ShowSeat {
    std::string seatLabel;
    std::unique_ptr<ShowSeatState> state;
    std::string heldBy; // userId currently holding, if any
    std::mutex mtx;     // protects the state transition itself — the actual concurrency guard

public:
    explicit ShowSeat(std::string label);

    const std::string& getLabel() const { return seatLabel; }
    void setState(std::unique_ptr<ShowSeatState> s) { state = std::move(s); }
    std::string currentStateName() const { return state->name(); }
    void setHeldBy(const std::string& userId) { heldBy = userId; }
    const std::string& getHeldBy() const { return heldBy; }

    // The lock ensures only one thread can execute a hold/confirm/release transition
    // at a time for THIS seat — the actual mechanism preventing double-booking.
    bool hold(const std::string& userId) {
        std::lock_guard<std::mutex> lock(mtx);
        return state->hold(*this, userId);
    }
    bool confirm() {
        std::lock_guard<std::mutex> lock(mtx);
        return state->confirm(*this);
    }
    void release() {
        std::lock_guard<std::mutex> lock(mtx);
        state->release(*this);
    }
};

class AvailableState : public ShowSeatState {
public:
    bool hold(ShowSeat& seat, const std::string& userId) override;
    std::string name() const override { return "Available"; }
};

class HeldState : public ShowSeatState {
public:
    bool hold(ShowSeat& seat, const std::string& userId) override {
        std::cout << "Seat " << seat.getLabel() << " already held by " << seat.getHeldBy() << "\n";
        return false; // rejected — someone else already holds it
    }
    bool confirm(ShowSeat& seat) override;
    void release(ShowSeat& seat) override; // called by HoldTimer on expiry, or user backs out
    std::string name() const override { return "Held"; }
};

class BookedState : public ShowSeatState {
public:
    std::string name() const override { return "Booked"; }
    // no hold/confirm valid from here; release() could be extended for cancellation flow
};

ShowSeat::ShowSeat(std::string label) : seatLabel(std::move(label)) {
    state = std::make_unique<AvailableState>();
}

bool AvailableState::hold(ShowSeat& seat, const std::string& userId) {
    seat.setHeldBy(userId);
    seat.setState(std::make_unique<HeldState>());
    std::cout << "Seat " << seat.getLabel() << " held by " << userId << "\n";
    return true;
}

bool HeldState::confirm(ShowSeat& seat) {
    seat.setState(std::make_unique<BookedState>());
    std::cout << "Seat " << seat.getLabel() << " booked (confirmed for " << seat.getHeldBy() << ")\n";
    return true;
}

void HeldState::release(ShowSeat& seat) {
    std::cout << "Seat " << seat.getLabel() << " hold released (timeout or user cancelled)\n";
    seat.setHeldBy("");
    seat.setState(std::make_unique<AvailableState>());
}

// ---------- Booking ----------
class Booking {
    std::string userId;
    std::vector<ShowSeat*> seats;
    double amountPaid;
    std::vector<BookingObserver*> observers;

public:
    Booking(std::string user, std::vector<ShowSeat*> s, double amount)
        : userId(std::move(user)), seats(std::move(s)), amountPaid(amount) {}

    const std::string& getUserId() const { return userId; }
    double getAmount() const { return amountPaid; }

    void addObserver(BookingObserver* o) { observers.push_back(o); }
    void notifyObservers() { for (auto* o : observers) o->update(*this); }
};

void NotificationService::update(const Booking& booking) {
    std::cout << "[Notify] Booking confirmed for " << booking.getUserId()
              << ", amount $" << booking.getAmount() << "\n";
}

// ---------- BookingSystem (Facade) ----------
class BookingSystem {
    PaymentProcessor payment;
    NotificationService notifier;

public:
    explicit BookingSystem(std::unique_ptr<PaymentStrategy> strat) : payment(std::move(strat)) {}

    bool holdSeats(std::vector<ShowSeat*>& seats, const std::string& userId) {
        std::vector<ShowSeat*> heldSoFar;
        for (auto* seat : seats) {
            if (!seat->hold(userId)) {
                // rollback any holds already acquired in this attempt
                for (auto* h : heldSoFar) h->release();
                std::cout << "Could not hold all requested seats — released partial holds\n";
                return false;
            }
            heldSoFar.push_back(seat);
        }
        return true;
    }

    std::unique_ptr<Booking> confirmBooking(std::vector<ShowSeat*>& seats, const std::string& userId, double amount) {
        if (!payment.charge(amount)) {
            std::cout << "Payment failed — releasing held seats\n";
            for (auto* seat : seats) seat->release();
            return nullptr;
        }
        for (auto* seat : seats) seat->confirm();
        auto booking = std::make_unique<Booking>(userId, seats, amount);
        booking->addObserver(&notifier);
        booking->notifyObservers();
        return booking;
    }
};

// ---------- Demo ----------
int main() {
    ShowSeat seatA1("A1");
    ShowSeat seatA2("A2");
    std::vector<ShowSeat*> selected = {&seatA1, &seatA2};

    BookingSystem system(std::make_unique<CardPayment>());

    // User A successfully holds and books
    if (system.holdSeats(selected, "userA")) {
        auto booking = system.confirmBooking(selected, "userA", 25.00);
    }

    // User B attempts the same seat after it's already booked — will fail at hold()
    ShowSeat& sameSeat = seatA1;
    if (!sameSeat.hold("userB")) {
        std::cout << "User B could not book seat A1 — already booked\n";
    }

    return 0;
}
```

---

## 10. Trade-offs

- **In-process `std::mutex` per seat vs a distributed lock (Redis)** — the demo uses a simple mutex to guard each `ShowSeat`'s state transition, which works for a single-process simulation; a real, horizontally-scaled booking system needs a **distributed** lock or an atomic conditional database update (`UPDATE show_seats SET state='HELD' WHERE id=? AND state='AVAILABLE'`) since multiple server instances handle concurrent requests — see [[redis]]'s `SET NX` pattern and [[sharding]] for scaling the underlying seat data itself.
- **Optimistic (DB conditional update) vs pessimistic (explicit lock) concurrency control** — this design leans pessimistic (lock, then check-and-transition); an alternative is optimistic concurrency (read seat state, attempt a versioned conditional update, retry on conflict — see [[transactions]]' OCC) which avoids holding a lock but requires retry logic on contention. Pessimistic locking is usually preferred here since seat contention is common for popular shows and retries under optimistic control could thrash.
- **Hold-then-pay vs pay-then-allocate** — holding a seat before payment (as designed) gives users a fair window to complete checkout without losing their selection to another user, but requires a `HoldTimer` and the complexity of releasing abandoned holds; a simpler "first to pay wins, no hold" model avoids that complexity but creates a frustrating UX where users can complete payment only to learn seats are gone — the hold pattern is the near-universal real-world choice despite the added complexity.
- **Central SeatLockManager vs per-show sharded locking** — for extremely popular shows (a single blockbuster's opening night across one theater), lock contention concentrates on that one show's seats; sharding the lock/seat data by show ID (so different shows' contention doesn't share infrastructure) helps horizontal scalability (see [[sharding]], [[scalability]]).
- **Automatic hold release via timer vs requiring explicit user cancellation** — automatic expiry (as designed) is essential for fairness (an abandoned checkout shouldn't lock a seat forever), but the timeout duration is a real product trade-off: too short frustrates slow users mid-payment, too long frustrates other users who see seats as unavailable that are actually abandoned.

---

## 11. Interview Questions

- **How do you guarantee a seat is never booked by two users at the same time?** The seat's state transition (`Available → Held`) must be **atomic** with respect to concurrent attempts — either via a database-level conditional update (`WHERE state = 'AVAILABLE'`, which naturally fails for the second concurrent request) or a distributed lock (e.g. Redis `SET NX`, see [[redis]]) acquired before the transition. The [[state]] pattern here defines _what_ transitions are valid, but the _atomicity_ of the check-and-transition against concurrent callers is what actually prevents double-booking — this distinction is the single most important thing to get right in this problem.
- **Why use a temporary "Held" state instead of directly booking (and charging) as soon as a user selects seats?** Payment can take time (entering card details, 3D-secure flows) — if seats were only reserved at the moment of successful payment, another user could book the same seats in the interim, leading to a frustrating "seats taken" failure _after_ the first user already committed to buying. A short hold window gives the first user exclusivity while they complete checkout, at the cost of needing hold-expiry logic for abandoned attempts.
- **How would you choose and tune the hold-timeout duration?** Balance user experience (enough time to enter payment details without feeling rushed) against seat availability for others (a shorter timeout frees up abandoned holds faster during high-demand periods) — many real systems use something like 5-10 minutes, sometimes dynamically shortened during extremely high-demand releases.
- **What happens if payment succeeds but the subsequent `confirm()` call fails (e.g. a crash between the two)?** This is the same "debit succeeded, downstream step failed" problem seen in the [[atm]] and [[food-delivery]] designs — the system needs a durable record of "payment succeeded, confirmation pending" (rather than performing both as independent, unlogged steps) so a recovery process can detect and complete (or refund and release) an interrupted booking rather than silently leaving a paid-for seat stuck in `HeldState` forever.
- **How would you scale this system for a simultaneous nationwide release of a blockbuster's tickets?** Shard seat/lock data by show (so different shows' seat contention is handled by different partitions, see [[sharding]]), scale the booking service horizontally behind a load balancer ([[load-balancing]]), and consider a **virtual waiting room / queueing** pattern at the edge (rate-limiting entry into the actual booking flow, see [[rate-limiting]]) so the seat-hold contention itself doesn't get overwhelmed by an initial traffic spike larger than the seat count.
- **How is this problem similar to the ATM's cash-dispensing correctness concern?** Both require that a "reserve/commit" pair of operations (debit-then-dispense in the ATM; hold-then-confirm here) either both succeed or the first is compensated/rolled back if the second fails — the same atomicity/compensating-action reasoning from [[acid]] applies directly, just with seats and payment instead of cash and account balance.

---

## Related Notes

- [[atm]]
- [[elevator]]
- [[food-delivery]]
- [[library]]
- [[state]]
- [[strategy]]
- [[observer]]
- [[facade]]
- [[proxy]]
- [[singleton]]
- [[transactions]]
- [[isolation-levels]]
- [[redis]]
- [[sharding]]
- [[rate-limiting]]
- [[load-balancing]]
- [[acid]]