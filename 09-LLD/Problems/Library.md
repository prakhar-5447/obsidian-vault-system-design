# Library Management System — Low Level Design

## 1. Requirements

### Functional

- Member can search the catalog (by title, author, ISBN, genre)
- Member can check out a book, and return it later
- Member can reserve a book that's currently checked out (placed in a queue for that title)
- System tracks due dates and calculates overdue fines
- System enforces a maximum number of books a member can hold at once
- Librarian can add/remove books (and copies of a title) from the catalog
- Librarian can register new members, suspend a member (e.g. for unpaid fines or lost books)
- System sends notifications for: due-date reminders, overdue fines, reserved book now available
- Support multiple physical **copies** of the same title (a "Book" is a title; a "BookItem" is one physical copy with its own barcode/ID and status)

### Non-Functional

- **Consistency** — a single physical copy must never be checked out to two members simultaneously, even under concurrent requests (ties to [[consistency]], [[transactions]])
- **Availability** — catalog search and member self-service (checkout/return/reserve) should remain usable even during peak hours (e.g. semester start at a university library)
- **Extensibility** — new item types (DVDs, magazines, e-books, equipment loans) should be addable without restructuring the core lending model
- **Auditability** — every checkout/return/fine transaction should be traceable for dispute resolution and reporting
- **Fairness** — a reservation queue for a popular title should be served in order (first-reserved, first-served) once a copy becomes available

---

## 2. Actors

- **Member** — searches catalog, checks out/returns/reserves books, pays fines
- **Librarian** — manages catalog (add/remove books/copies), manages members (register/suspend), processes checkouts/returns at the desk
- **System** (internal actor) — tracks due dates, computes fines, sends notifications, manages the reservation queue

---

## 3. Use Cases

- Search catalog by title/author/genre/ISBN
- Check out a book (self-service or via librarian)
- Return a book
- Reserve a currently-unavailable title; get notified when a copy becomes free
- Renew a checked-out book (extend due date, if not reserved by someone else)
- Calculate and pay an overdue fine
- Add/remove a book title or a specific physical copy from the catalog
- Register a new member; suspend a member (e.g. exceeded fine threshold, lost item)
- Enforce checkout limits (e.g. max 5 books per member at once)

---

## 4. Classes

|Class|Responsibility|
|---|---|
|**Library**|Top-level facade coordinating catalog search, checkout, return, reservation — see [[facade]]|
|**Catalog**|Holds the searchable collection of `Book` titles, indexed by title/author/ISBN/genre|
|**Book**|A title-level entity (metadata: title, author, ISBN, genre) — distinct from a physical copy|
|**BookItem**|One physical copy of a `Book`, with its own barcode/ID and a current [[state]] (Available, CheckedOut, Reserved, Lost)|
|**BookItemState (interface)**|Declares state-dependent operations (`checkout()`, `returnItem()`, `reserve()`, `markLost()`)|
|**AvailableState / CheckedOutState / ReservedState / LostState** (concrete states)|Each governs valid transitions for that copy's lifecycle|
|**Member**|Holds member info, current checked-out items, fines owed, suspension status|
|**Librarian**|A privileged actor that can modify the Catalog and Member records|
|**Checkout / Reservation**|Records a lending transaction: which BookItem, which Member, checkout/due date; Reservation additionally tracks queue position|
|**ReservationQueue**|Per-title FIFO queue of members waiting for a copy to become available|
|**FineCalculator (interface)**|Declares `calculateFine(checkout)` — a pluggable [[strategy]] so fine policy can vary (flat rate vs escalating vs member-type-based)|
|**NotificationService**|Observes BookItem/Checkout state changes and Member events, sends due-date/fine/availability notifications — see [[observer]]|
|**SearchStrategy (interface)**|Declares `search(catalog, query)` — different search algorithms (by title, by author, full-text) implement this — see [[strategy]]|

---

## 5. Relationships

- **Library "has-a" Catalog, has-many Member, has-a NotificationService** — composition; the facade coordinates all of them for checkout/return/reserve operations
- **Book "has-many" BookItem** — composition/aggregation; one title can have multiple physical copies, each tracked independently
- **BookItem "has-a" BookItemState** — composition; a copy delegates its behavior to its current state (State pattern, same structural approach as [[atm]], [[elevator]], and [[food-delivery]])
- **Member "has-many" Checkout** — association; a member's currently-held items and history
- **Book "has-a" ReservationQueue** — composition; each title (not each copy) has its own queue, since a reservation is for "the next available copy of this title," not a specific physical one
- **Library "uses-a" FineCalculator, uses-a SearchStrategy** — Strategy pattern; both the fine policy and the search algorithm are swappable without changing Library's coordinating logic
- **BookItem "is observed by" NotificationService** — Observer pattern; a copy becoming `Available` (e.g. after a return) notifies the service, which checks the ReservationQueue and notifies the next waiting member
- **Checkout "references" BookItem and Member** — association, created at checkout time, closed (but retained for history/audit) at return time

---

## 6. Design Patterns

- **[[state]]** — each `BookItem`'s lifecycle (`Available → CheckedOut → (Available or Reserved) → ...`, plus a `Lost` terminal-ish state reachable from most states) is modeled as a state machine, the same structural reasoning used in [[atm]], [[elevator]], and [[food-delivery]]. This makes invalid operations (e.g. checking out an already-`CheckedOut` copy) structurally impossible rather than an ad hoc check scattered across the codebase.
- **[[strategy]]** — used **twice**: (1) `FineCalculator` for the overdue-fine policy (flat per-day rate vs escalating rate vs waived for certain member types), and (2) `SearchStrategy` for catalog search (search-by-title vs search-by-author vs a full-text search across all fields) — both swappable without touching `Library`'s coordinating code.
- **[[observer]]** — `BookItem` notifies the `NotificationService` on every state change; when a returned copy transitions back to `Available`, the service checks that title's `ReservationQueue` and notifies the next member in line — decoupling "a copy became available" from "who needs to know and what happens next."
- **[[factory-method]]** — a `CheckoutFactory` (or similar) could create the correct kind of lending record if the system is extended to support different item types (books vs equipment vs e-books, each perhaps with different loan-period rules), keeping that decision out of the core checkout flow.
- **[[facade]]** — `Library` exposes simple, high-level methods (`checkout()`, `returnItem()`, `reserve()`, `search()`) that internally coordinate `Catalog`, `Member`, `BookItem` state transitions, and `NotificationService`, without exposing that orchestration to the client (e.g. a front-desk UI or self-checkout kiosk).
- **[[singleton]]** — `Catalog` and `NotificationService` are natural single-instance-per-library components; as with prior LLD problems, best implemented via dependency injection as singleton-scoped services rather than a classic `getInstance()` (see [[singleton]] trade-offs).

---

## 7. Class Diagram

```text
                    ┌────────────────────┐
                    │       Library         │ (Facade)
                    │ - catalog              │
                    │ - notificationService   │
                    │ - fineCalculator        │
                    │ - searchStrategy        │
                    │ + checkout(member, item) │
                    │ + returnItem(item)        │
                    │ + reserve(member, book)    │
                    │ + search(query)             │
                    └─────────┬──────────┘
                              │ coordinates
        ┌─────────────────────┼─────────────────────┐
        ▼                      ▼                       ▼
┌────────────────┐  ┌──────────────────────┐ ┌──────────────────────┐
│     Catalog       │  │  FineCalculator (if)   │ │ NotificationService    │
│ - books: List       │  │  + calculateFine(co)    │ │ (implements Observer)  │
│ + search(query)      │  └──────────────────────┘ └──────────────────────┘
└────────┬───────┘
         │ has-many
         ▼
┌────────────────┐
│      Book         │
│ - title, author,    │
│   isbn, genre        │
│ - items: List<BookItem>│
│ - reservationQueue     │
└────────┬───────┘
         │ has-many
         ▼
┌────────────────────┐        ┌──────────────────────┐
│      BookItem         │───────▶│ BookItemState (interface)│
│ - barcode               │        │ + checkout()               │
│ - state                   │        │ + returnItem()               │
│ - observers: List           │        │ + reserve()                    │
│ + notifyObservers()           │        │ + markLost()                    │
└──────────┬─────────────┘        └─────────┬─────────────┘
           │ notifies                          │ implemented by
           ▼                       ┌───────────┼──────────┬──────────────┐
 ┌──────────────────┐               ▼             ▼          ▼              ▼
 │ NotificationService │     ┌──────────────┐┌──────────────┐┌──────────────┐┌───────────┐
 └──────────────────┘     │AvailableState  ││CheckedOutState ││ReservedState  ││LostState   │
                            └──────────────┘└──────────────┘└──────────────┘└───────────┘

┌────────────────┐        ┌──────────────────┐
│     Member        │        │     Checkout        │
│ - checkouts: List   │───────▶│ - bookItem, member    │
│ - finesOwed           │        │ - checkoutDate, dueDate│
│ - suspended             │        └──────────────────┘
└────────────────┘

┌────────────────┐
│ ReservationQueue  │
│ - queue: FIFO<Member>│
│ + enqueue(member)     │
│ + notifyNext()         │
└────────────────┘
```

---

## 8. Sequence Flow

```text
Checkout with an existing reservation queue — happy path

Member A     Library        BookItem(state)   Catalog     FineCalculator   NotificationService   ReservationQueue
   │ returnItem(item) │                │              │                │                    │                │
   │──────────────────▶│                │              │                │                    │                │
   │                    │──returnItem()──▶│ (CheckedOutState│              │                    │                │
   │                    │                │  .returnItem())│              │                    │                │
   │                    │                │──calculateFine(checkout)──────▶│                    │                │
   │                    │                │◀──── fine amount (if any) ────│                    │                │
   │                    │                │ (transitions to │              │                    │                │
   │                    │                │  ReservedState,   │              │                    │                │
   │                    │                │  since queue non-empty)         │                    │                │
   │                    │                │──notifyObservers()──────────────┼───────────────────▶│                │
   │                    │                │              │                │──notifyNext(title)──────────────────▶│
   │                    │                │              │                │                    │◀── Member B ───│
   │                    │                │              │                │──sendNotification(MemberB, "available")│
   │◀── fine charged, return confirmed ──│              │                │                    │                │

... later ...

Member B     Library        BookItem(state)
   │ checkout(item)    │                │
   │──────────────────▶│                │
   │                    │──checkout()────▶│ (ReservedState.checkout():
   │                    │                │  verifies Member B is the
   │                    │                │  reservation holder, transitions
   │                    │                │  to CheckedOutState)
   │◀── checkout confirmed, due date set │

Failure branch: if a member OTHER than the reservation holder attempts
checkout() while the item is in ReservedState, the state rejects it —
reservations must be honored in FIFO order (see ReservationQueue),
preventing a walk-in checkout from bypassing someone who reserved first.
```

---

## 9. Code

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <deque>
#include <memory>
#include <optional>
#include <ctime>

// ---------- Forward declarations ----------
class BookItem;
class Member;

// ---------- Member ----------
class Member {
    int id;
    std::string name;
    double finesOwed = 0;
    bool suspended = false;

public:
    Member(int id_, std::string name_) : id(id_), name(std::move(name_)) {}
    int getId() const { return id; }
    const std::string& getName() const { return name; }
    void addFine(double amount) { finesOwed += amount; }
    double getFinesOwed() const { return finesOwed; }
    bool isSuspended() const { return suspended; }
    void suspend() { suspended = true; }
};

// ---------- Checkout record ----------
struct Checkout {
    BookItem* item;
    Member* member;
    time_t checkoutDate;
    time_t dueDate;
};

// ---------- Fine Calculator (Strategy) ----------
class FineCalculator {
public:
    virtual ~FineCalculator() = default;
    virtual double calculateFine(const Checkout& checkout, time_t returnDate) = 0;
};

class FlatRateFineCalculator : public FineCalculator {
    double perDayRate;
public:
    explicit FlatRateFineCalculator(double rate) : perDayRate(rate) {}
    double calculateFine(const Checkout& checkout, time_t returnDate) override {
        double overdueDays = std::max(0.0, difftime(returnDate, checkout.dueDate) / 86400.0);
        return overdueDays * perDayRate;
    }
};

// ---------- Notification Observer ----------
class NotificationService {
public:
    void notifyAvailable(Member& member, const std::string& title) {
        std::cout << "[Notify] " << member.getName() << ": '" << title << "' is now available for you\n";
    }
    void notifyFine(Member& member, double amount) {
        std::cout << "[Notify] " << member.getName() << ": fine charged $" << amount << "\n";
    }
};

// ---------- Reservation Queue (per title) ----------
class ReservationQueue {
    std::deque<Member*> queue;
public:
    void enqueue(Member* m) { queue.push_back(m); }
    bool empty() const { return queue.empty(); }
    Member* front() const { return queue.front(); }
    void pop() { queue.pop_front(); }
};

// ---------- BookItem States ----------
class BookItemState {
public:
    virtual ~BookItemState() = default;
    virtual void checkout(BookItem& item, Member& member) { std::cout << "Cannot checkout in current state\n"; }
    virtual void returnItem(BookItem& item) { std::cout << "Cannot return in current state\n"; }
    virtual void reserve(BookItem& item, Member& member) { std::cout << "Cannot reserve in current state\n"; }
    virtual void markLost(BookItem& item) { std::cout << "Cannot mark lost in current state\n"; }
    virtual std::string name() const = 0;
};

// ---------- BookItem (Context + Subject) ----------
class BookItem {
    std::string barcode;
    std::string title;
    std::unique_ptr<BookItemState> state;
    ReservationQueue reservationQueue;
    std::optional<Checkout> activeCheckout;
    FineCalculator& fineCalculator;
    NotificationService& notifier;

public:
    BookItem(std::string barcode_, std::string title_, FineCalculator& fc, NotificationService& ns)
        : barcode(std::move(barcode_)), title(std::move(title_)), fineCalculator(fc), notifier(ns);

    // placeholder constructor body defined below due to member init order
};

// NOTE: constructor body separated for clarity of state assignment
BookItem::BookItem(std::string barcode_, std::string title_, FineCalculator& fc, NotificationService& ns)
    : barcode(std::move(barcode_)), title(std::move(title_)), fineCalculator(fc), notifier(ns) {}

// (Continuing outside the class due to the placeholder above — see full listing note.)
```

> **Note on the C++ listing above:** the `BookItem` class needs its state initialized inline (e.g. `state = std::make_unique<AvailableState>();` in the constructor body) and the concrete state classes defined afterward, since they reference `BookItem` methods. For clarity and brevity in an interview setting, the pattern is shown below as a single consolidated, compilable sketch instead of the fragmented version above:

```cpp
#include <iostream>
#include <string>
#include <deque>
#include <memory>
#include <optional>
#include <ctime>

class Member {
    std::string name;
    double finesOwed = 0;
public:
    explicit Member(std::string name_) : name(std::move(name_)) {}
    const std::string& getName() const { return name; }
    void addFine(double amount) { finesOwed += amount; }
};

struct Checkout {
    Member* member;
    time_t dueDate;
};

class NotificationService {
public:
    void notifyAvailable(Member& member, const std::string& title) {
        std::cout << "[Notify] " << member.getName() << ": '" << title << "' now available\n";
    }
    void notifyFine(Member& member, double amount) {
        std::cout << "[Notify] " << member.getName() << ": fine charged $" << amount << "\n";
        member.addFine(amount);
    }
};

class FineCalculator {
public:
    virtual ~FineCalculator() = default;
    virtual double calculateFine(const Checkout& co, time_t returnDate) = 0;
};

class FlatRateFineCalculator : public FineCalculator {
    double perDayRate;
public:
    explicit FlatRateFineCalculator(double rate) : perDayRate(rate) {}
    double calculateFine(const Checkout& co, time_t returnDate) override {
        double overdueDays = std::max(0.0, difftime(returnDate, co.dueDate) / 86400.0);
        return overdueDays * perDayRate;
    }
};

class ReservationQueue {
    std::deque<Member*> queue;
public:
    void enqueue(Member* m) { queue.push_back(m); }
    bool empty() const { return queue.empty(); }
    Member* popFront() { Member* m = queue.front(); queue.pop_front(); return m; }
};

class BookItem;

class BookItemState {
public:
    virtual ~BookItemState() = default;
    virtual void checkout(BookItem& item, Member& member) { std::cout << "Cannot checkout now\n"; }
    virtual void returnItem(BookItem& item, time_t returnDate) { std::cout << "Cannot return now\n"; }
    virtual void reserve(BookItem& item, Member& member) { std::cout << "Cannot reserve now\n"; }
    virtual std::string name() const = 0;
};

class BookItem {
    std::string title;
    std::unique_ptr<BookItemState> state;
    ReservationQueue reservationQueue;
    std::optional<Checkout> activeCheckout;
    FineCalculator& fineCalc;
    NotificationService& notifier;

public:
    BookItem(std::string title_, FineCalculator& fc, NotificationService& ns);

    const std::string& getTitle() const { return title; }
    ReservationQueue& getQueue() { return reservationQueue; }
    FineCalculator& getFineCalc() { return fineCalc; }
    NotificationService& getNotifier() { return notifier; }
    std::optional<Checkout>& getActiveCheckout() { return activeCheckout; }

    void setState(std::unique_ptr<BookItemState> s) { state = std::move(s); }
    std::string currentStateName() const { return state->name(); }

    void checkout(Member& member) { state->checkout(*this, member); }
    void returnItem(time_t returnDate) { state->returnItem(*this, returnDate); }
    void reserve(Member& member) { state->reserve(*this, member); }
};

class AvailableState : public BookItemState {
public:
    void checkout(BookItem& item, Member& member) override;
    void reserve(BookItem& item, Member& member) override {
        item.getQueue().enqueue(&member); // reserve while available == "hold it for me now"
    }
    std::string name() const override { return "Available"; }
};

class ReservedState : public BookItemState {
public:
    void checkout(BookItem& item, Member& member) override; // only the front of queue may checkout
    std::string name() const override { return "Reserved"; }
};

class CheckedOutState : public BookItemState {
public:
    void returnItem(BookItem& item, time_t returnDate) override;
    void reserve(BookItem& item, Member& member) override {
        item.getQueue().enqueue(&member); // wait in line
    }
    std::string name() const override { return "CheckedOut"; }
};

BookItem::BookItem(std::string title_, FineCalculator& fc, NotificationService& ns)
    : title(std::move(title_)), fineCalc(fc), notifier(ns) {
    state = std::make_unique<AvailableState>();
}

void AvailableState::checkout(BookItem& item, Member& member) {
    time_t now = time(nullptr);
    time_t due = now + 14 * 86400; // 14-day loan period
    item.getActiveCheckout() = Checkout{&member, due};
    item.setState(std::make_unique<CheckedOutState>());
    std::cout << "Checked out '" << item.getTitle() << "' to " << member.getName() << "\n";
}

void ReservedState::checkout(BookItem& item, Member& member) {
    // simplified: assumes caller already verified `member` is the front of the queue
    item.getQueue().popFront();
    time_t now = time(nullptr);
    time_t due = now + 14 * 86400;
    item.getActiveCheckout() = Checkout{&member, due};
    item.setState(std::make_unique<CheckedOutState>());
    std::cout << "Reserved checkout completed for " << member.getName() << "\n";
}

void CheckedOutState::returnItem(BookItem& item, time_t returnDate) {
    auto& checkout = item.getActiveCheckout();
    if (checkout) {
        double fine = item.getFineCalc().calculateFine(*checkout, returnDate);
        if (fine > 0) item.getNotifier().notifyFine(*checkout->member, fine);
        item.getActiveCheckout().reset();
    }
    if (!item.getQueue().empty()) {
        Member* next = item.getQueue().popFront();
        item.getNotifier().notifyAvailable(*next, item.getTitle());
        item.setState(std::make_unique<ReservedState>());
    } else {
        item.setState(std::make_unique<AvailableState>());
    }
    std::cout << "'" << item.getTitle() << "' returned\n";
}

// ---------- Demo ----------
int main() {
    FlatRateFineCalculator fineCalc(0.50); // $0.50/day
    NotificationService notifier;

    BookItem book("Clean Code", fineCalc, notifier);
    Member alice("Alice");
    Member bob("Bob");

    book.checkout(alice);                 // Alice checks it out
    book.reserve(bob);                    // Bob reserves it (queues behind Alice's checkout)

    time_t lateReturn = time(nullptr) + 20 * 86400; // simulate a 6-day-overdue return
    book.returnItem(lateReturn);           // Alice returns late -> fine charged, Bob notified

    book.checkout(bob);                    // Bob completes his reserved checkout

    return 0;
}
```

---

## 10. Trade-offs

- **Per-title reservation queue vs per-copy reservation** — reserving "the next available copy of this title" (as modeled here) is simpler and fairer for members (any copy will do), but means the system can't let a member reserve one _specific_ physical copy (rarely needed, but relevant for e.g. a signed/rare edition) — a reasonable simplification for most libraries.
- **State pattern per BookItem vs a status enum on Book** — modeling state per physical copy (not per title) is more correct (different copies of the same title can be in different states simultaneously), but means every operation must resolve down to a specific `BookItem`, not just a `Book` — a bit more indirection than a naive single-status-per-title model, but necessary for correctness with multiple copies.
- **FlatRateFineCalculator vs escalating/tiered fine policies** — a flat per-day rate is simple and predictable but doesn't discourage very long overdue periods as strongly as an escalating rate would; making this a [[strategy]] means the policy can be swapped per library branch or member type without touching the checkout/return flow.
- **Synchronous fine calculation and notification vs asynchronous** — the demo calculates and reports fines synchronously within `returnItem()`; a production system serving many concurrent front-desk terminals might instead publish a "book returned" event (see [[event-driven-architecture]], [[message-queues]]) and let a separate fine-processing consumer handle calculation and notification asynchronously, improving front-desk responsiveness.
- **In-memory reservation queue vs persisted queue** — the demo's `ReservationQueue` is a simple in-memory deque; a real system must persist this (in the same database transaction as the checkout/return, see [[transactions]]) so a server restart doesn't silently lose a member's place in line.

---

## 11. Interview Questions

- **How do you prevent two members from checking out the same physical copy simultaneously?** The `BookItem`'s State pattern makes this a natural single point of enforcement — `checkout()` is only valid from `AvailableState` (or `ReservedState` for the correct member); once a checkout occurs, the state transitions to `CheckedOutState`, and any concurrent attempt would need to acquire a lock (row-level lock or optimistic version check, see [[transactions]], [[isolation-levels]]) around the state check-and-transition to avoid a race where two requests both see `AvailableState` before either commits its transition.
- **Why model `Book` and `BookItem` as separate classes instead of one class with a `status` field and a `copies` count?** Because different physical copies of the same title can be in different states at once (one checked out, one available, one lost) — a single `Book` with just a count can't represent that; `Book` should model title-level metadata (author, ISBN, genre) and `BookItem` should model per-copy state, matching real-world reality where copies are independently tracked (barcode/RFID).
- **How would you handle a member trying to renew a book that someone else has reserved?** The renewal operation should check whether the title's `ReservationQueue` is non-empty before extending the due date; if there's a pending reservation, renewal should be rejected (or capped) so the item can transition to `ReservedState` and reach the waiting member as expected, once returned, rather than being indefinitely re-extended by the current holder.
- **How would you design the fine calculation to support different rates for different member types (e.g. students vs faculty)?** Keep `FineCalculator` as a [[strategy]], but parameterize it further by member type (or have `Library` select a different `FineCalculator` instance per member category) — this avoids hardcoding member-type-specific logic into the checkout/return flow itself.
- **How would you extend this design to support other item types (DVDs, laptops, study rooms) with different loan periods and fine rules?** Introduce a common `LendableItem` interface/abstract base that `BookItem` (and new types like `DvdItem`, `EquipmentItem`) implement, each configuring its own loan period and possibly its own `FineCalculator` — the `Library` facade and State-pattern machinery generalize naturally since they were written against the abstraction rather than `BookItem` specifically.
- **What happens if a book is marked "Lost" while a member is waiting in the reservation queue for it?** The transition to a `LostState` should trigger a notification to everyone currently in that title's `ReservationQueue` (via the same [[observer]] mechanism used for availability), informing them the copy is unavailable and, if other copies of the same title exist, re-routing their reservation to the title's overall queue rather than the specific lost copy.

---

## Related Notes

- [[atm]]
- [[elevator]]
- [[food-delivery]]
- [[state]]
- [[strategy]]
- [[observer]]
- [[facade]]
- [[factory]]
- [[singleton]]
- [[transactions]]
- [[isolation-levels]]
- [[event-driven-architecture]]