# Parking Lot System — Low Level Design

## 1. Requirements

### Functional

- Vehicle enters the parking lot and is assigned an available spot matching its type (motorcycle, car, bus/truck)
- System issues a parking ticket at entry, recording entry time and assigned spot
- Vehicle exits by presenting the ticket; system calculates the parking fee based on duration and vehicle type
- Support multiple spot types/sizes: **Motorcycle**, **Compact**, **Large** — a larger vehicle can use a larger spot if its own type's spots are full (but not vice versa)
- Support multiple floors, each with its own spots
- Display/track real-time availability (free spot count per type, per floor)
- Support multiple entry/exit gates
- Handle "lot full" for a given vehicle type gracefully (reject entry or redirect)
- Support different payment methods at exit

### Non-Functional

- **Consistency** — a single spot must never be assigned to two vehicles simultaneously, even with multiple entry gates operating concurrently (ties to [[transactions]], [[isolation-levels]])
- **Availability** — entry/exit processing should keep working even if one gate's hardware has an issue (doesn't block other gates)
- **Scalability** — should work whether the lot has 50 spots or 5,000 across many floors ([[scalability]])
- **Low latency** — spot assignment at entry should be near-instant to avoid a queue building up at the gate (ties to [[latency]])
- **Extensibility** — new vehicle types, new spot types, or new pricing schemes should be addable without restructuring the core allocation logic
- **Reliability** — a lost/corrupted ticket shouldn't leave a spot permanently marked occupied with no way to reconcile it (ties to [[reliability]])

---

## 2. Actors

- **Driver** — enters/exits with a vehicle, receives a ticket, pays the fee
- **Parking Attendant / Admin** — configures floors/spots, handles exceptions (lost ticket, manual override)
- **Entry Gate / Exit Gate** (system-internal actors) — hardware endpoints that trigger spot assignment and fee calculation respectively
- **Payment Gateway** (external system) — processes the exit payment

---

## 3. Use Cases

- Vehicle arrives at an entry gate, is assigned a spot, and receives a ticket
- Vehicle presents a ticket at an exit gate, fee is calculated, payment is processed, spot is freed
- System rejects entry when no suitable spot is available for that vehicle type
- Admin adds/removes a floor or spot
- System reports current availability by floor and vehicle type
- Handle a vehicle whose own spot type is full, by allocating a larger spot type instead

---

## 4. Classes

|Class|Responsibility|
|---|---|
|**ParkingLot**|Top-level facade coordinating entry, exit, and spot lookup — see [[facade]]|
|**ParkingFloor**|Holds the spots for one floor, tracks per-type availability counts|
|**ParkingSpot (abstract)**|A single spot; subclasses `MotorcycleSpot`, `CompactSpot`, `LargeSpot` model size/type-specific capacity rules|
|**SpotState (interface)**|Declares state-dependent operations (`assign(vehicle)`, `free()`)|
|**AvailableState / OccupiedState** (concrete states)|Govern valid transitions for a spot|
|**Vehicle (abstract)**|Base for `Motorcycle`, `Car`, `Bus` — each declares which spot type(s) it can use|
|**Ticket**|Issued at entry; records vehicle, assigned spot, entry time; presented at exit|
|**SpotAllocationStrategy (interface)**|Declares `findSpot(vehicle, floors)` — pluggable allocation algorithm — see [[strategy]]|
|**NearestSpotStrategy / BestFitStrategy** (concrete strategies)|Concrete allocation algorithms|
|**FeeCalculator (interface)**|Declares `calculateFee(ticket, exitTime)` — pluggable pricing (flat-rate, hourly-tiered, vehicle-type-based) — see [[strategy]]|
|**EntryGate / ExitGate**|Hardware-facing endpoints; EntryGate requests a spot + issues a ticket, ExitGate validates the ticket + triggers fee calculation and payment|
|**PaymentProcessor (interface / proxy)**|Gateway to the external payment provider — a [[proxy]]|
|**DisplayBoard (observer)**|Observes spot state changes to show live availability — see [[observer]]|

---

## 5. Relationships

- **ParkingLot "has-many" ParkingFloor** — composition
- **ParkingFloor "has-many" ParkingSpot** — composition
- **ParkingSpot "has-a" SpotState** — composition; delegates behavior to its current state (State pattern, same structural approach as [[atm]], [[elevator]], [[food-delivery]], [[library]], and [[movie-booking]])
- **ParkingLot "uses-a" SpotAllocationStrategy, uses-a FeeCalculator** — Strategy pattern; both allocation and pricing are swappable independent of the core entry/exit flow
- **Ticket "references" Vehicle and ParkingSpot** — association, created at entry, closed at exit
- **Vehicle (abstract) "is extended by"** `Motorcycle`, `Car`, `Bus` — inheritance; each declares which `ParkingSpot` subtype(s) it's compatible with (a `Bus` needs a `LargeSpot`; a `Motorcycle` can use a `MotorcycleSpot` or, if full, a larger one)
- **ParkingSpot "is observed by" DisplayBoard** — Observer pattern; spot occupancy changes update the live availability display without ParkingSpot knowing who's watching
- **EntryGate "uses" SpotAllocationStrategy (via ParkingLot)**, **ExitGate "uses" FeeCalculator and PaymentProcessor (via ParkingLot)** — both gates interact with the system only through the ParkingLot facade

---

## 6. Design Patterns

- **[[state]]** — each `ParkingSpot`'s lifecycle (`Available ↔ Occupied`) is modeled as a state machine — simpler than prior LLD problems (only two states, no complex intermediate stages), but still valuable for consistency with the rest of the vault's approach and for cleanly blocking invalid operations (e.g. `free()` on an already-`Available` spot).
- **[[strategy]]** — used **twice**: (1) `SpotAllocationStrategy` for spot assignment (nearest spot to the entry gate vs best-fit — assign the smallest spot type that still fits the vehicle, to preserve larger spots for vehicles that actually need them), and (2) `FeeCalculator` for pricing (flat hourly rate vs tiered/progressive rate vs vehicle-type multiplier) — both swappable without touching `ParkingLot`'s core flow.
- **[[observer]]** — `ParkingSpot` (or `ParkingFloor`, aggregating its spots) notifies a `DisplayBoard` whenever occupancy changes, decoupling "a spot's state changed" from "who needs to update a live availability sign."
- **[[factory-method]]** — a `VehicleFactory`/`TicketFactory` can construct the correct `Vehicle` subtype from a license-plate scan or gate input, and a `SpotAllocationStrategy` implementation can itself be selected via a factory based on lot configuration (e.g. a large mall lot might use `BestFitStrategy`, a small lot `NearestSpotStrategy`).
- **[[facade]]** — `ParkingLot` exposes simple high-level operations (`parkVehicle()`, `unparkVehicle()`, `getAvailability()`) that internally coordinate floors, spots, allocation strategy, fee calculation, and payment, hiding that orchestration from the gate hardware/client code.
- **[[singleton]]** — the `ParkingLot` instance itself (and its availability-tracking structures) is a natural single-instance-per-facility component, best implemented via dependency injection as a singleton-scoped service (see [[singleton]] trade-offs) rather than a classic `getInstance()`.

---

## 7. Class Diagram

```text
                    ┌────────────────────────┐
                    │       ParkingLot           │ (Facade)
                    │ - floors: List                │
                    │ - allocationStrategy             │
                    │ - feeCalculator                     │
                    │ - paymentProcessor                    │
                    │ + parkVehicle(vehicle): Ticket           │
                    │ + unparkVehicle(ticket, payment)           │
                    │ + getAvailability()                          │
                    └─────────┬──────────────────────┘
                              │ has-many
                              ▼
                    ┌────────────────────┐
                    │    ParkingFloor       │
                    │ - spots: List           │
                    │ + availableCount(type)   │
                    └─────────┬──────────┘
                              │ has-many
                              ▼
                    ┌────────────────────┐        ┌──────────────────────┐
                    │  ParkingSpot (abstract)│───────▶│  SpotState (interface)  │
                    │ - state                  │        │ + assign(vehicle)          │
                    │ - observers: List           │        │ + free()                      │
                    │ + notifyObservers()          │        └─────────┬─────────────┘
                    └──────────┬───────────┘                  │ implemented by
           ┌────────────┬───────┴────────┬───────────┐  ┌───────┴────────┐
           ▼             ▼                 ▼           ▼           ▼
   ┌──────────────┐┌──────────────┐┌──────────────┐  ┌──────────┐┌──────────────┐
   │MotorcycleSpot ││CompactSpot    ││LargeSpot        │  │AvailableState│OccupiedState│
   └──────────────┘└──────────────┘└──────────────┘  └──────────┘└──────────────┘

┌────────────────┐        ┌──────────────────────┐
│ Vehicle (abstract) │        │ SpotAllocationStrategy(if)│
│ + compatibleSpotTypes()│        │ + findSpot(vehicle, floors) │
└────────┬───────┘        └─────────┬─────────────┘
   ┌─────┼──────┐              ┌────┴─────┐
   ▼      ▼        ▼            ▼            ▼
┌─────┐┌─────┐┌─────┐  ┌──────────────┐┌──────────────┐
│Motor-││ Car ││ Bus  │  │NearestSpot     ││BestFitStrategy │
│cycle │└─────┘└─────┘  │Strategy         │└──────────────┘
└─────┘                 └──────────────┘

┌────────────────┐        ┌──────────────────┐
│     Ticket         │        │  FeeCalculator (if) │
│ - vehicle, spot       │        │ + calculateFee(ticket, exitTime)│
│ - entryTime             │        └──────────────────┘
└────────────────┘

┌────────────────┐        ┌──────────────────┐
│  EntryGate         │        │    ExitGate          │
│ + processEntry()      │        │ + processExit(ticket) │
└────────────────┘        └──────────────────┘

┌────────────────┐
│ PaymentProcessor    │  (Proxy — same role as in [[food-delivery]], [[movie-booking]])
│ + charge(amount)       │
└────────────────┘
```

---

## 8. Sequence Flow

```text
Entry — Happy Path

Driver     EntryGate      ParkingLot      SpotAllocationStrategy   ParkingSpot(state)   DisplayBoard
   │ arrives     │               │                    │                       │              │
   │─────────────▶│               │                    │                       │              │
   │              │──processEntry(vehicle)──▶│           │                       │              │
   │              │               │──findSpot(vehicle, floors)──▶│                │              │
   │              │               │                    │ (scans floors for a    │              │
   │              │               │                    │  compatible AvailableState│            │
   │              │               │                    │  spot, nearest/best-fit)  │            │
   │              │               │◀──── Spot #F2-C14 ─│                       │              │
   │              │               │──assign(vehicle)───┼──────────────────────▶│              │
   │              │               │                    │                       │ (Available    │
   │              │               │                    │                       │  ->Occupied)  │
   │              │               │                    │                       │──notify()────▶│
   │              │               │ (creates Ticket:     │                       │              │
   │              │               │  vehicle, spot,        │                       │              │
   │              │               │  entryTime=now)          │                       │              │
   │◀── ticket ───│◀──── ticket ──│                    │                       │              │

Exit — Happy Path

Driver     ExitGate       ParkingLot      FeeCalculator   PaymentProcessor   ParkingSpot(state)   DisplayBoard
   │ presents ticket│              │                │                 │                    │              │
   │────────────────▶│              │                │                 │                    │              │
   │                 │──processExit(ticket)──▶│        │                 │                    │              │
   │                 │              │──calculateFee(ticket, now)──▶│     │                    │              │
   │                 │              │◀──── $12.50 ────│                 │                    │              │
   │                 │              │──charge($12.50)─┼────────────────▶│                    │              │
   │                 │              │◀──── success ───┼─────────────────│                    │              │
   │                 │              │──free()─────────┼─────────────────┼───────────────────▶│              │
   │                 │              │                │                 │ (Occupied->Available)│              │
   │                 │              │                │                 │──notify()───────────┼─────────────▶│
   │◀── receipt, gate opens ────────│              │                │                    │              │

Failure branch: if the lot is full for the vehicle's compatible spot types
(SpotAllocationStrategy.findSpot() returns none), EntryGate rejects entry
before a Ticket is ever created — no partial state is left behind.
```

---

## 9. Code

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>
#include <ctime>
#include <limits>

// ---------- Vehicle hierarchy ----------
enum class SpotType { MOTORCYCLE, COMPACT, LARGE };

class Vehicle {
public:
    virtual ~Vehicle() = default;
    virtual std::vector<SpotType> compatibleSpotTypes() const = 0; // ordered: preferred first
    virtual std::string licensePlate() const = 0;
};

class Motorcycle : public Vehicle {
    std::string plate;
public:
    explicit Motorcycle(std::string p) : plate(std::move(p)) {}
    std::vector<SpotType> compatibleSpotTypes() const override {
        return {SpotType::MOTORCYCLE, SpotType::COMPACT, SpotType::LARGE}; // can upgrade if full
    }
    std::string licensePlate() const override { return plate; }
};

class Car : public Vehicle {
    std::string plate;
public:
    explicit Car(std::string p) : plate(std::move(p)) {}
    std::vector<SpotType> compatibleSpotTypes() const override {
        return {SpotType::COMPACT, SpotType::LARGE};
    }
    std::string licensePlate() const override { return plate; }
};

class Bus : public Vehicle {
    std::string plate;
public:
    explicit Bus(std::string p) : plate(std::move(p)) {}
    std::vector<SpotType> compatibleSpotTypes() const override {
        return {SpotType::LARGE}; // only fits large spots
    }
    std::string licensePlate() const override { return plate; }
};

// ---------- Spot State ----------
class ParkingSpot;

class SpotState {
public:
    virtual ~SpotState() = default;
    virtual bool assign(ParkingSpot& spot, Vehicle* vehicle) { return false; }
    virtual void free(ParkingSpot& spot) { std::cout << "Cannot free — already available\n"; }
    virtual std::string name() const = 0;
};

// ---------- Observer ----------
class DisplayBoard {
public:
    void update(const std::string& spotId, bool nowOccupied) {
        std::cout << "[Display] Spot " << spotId << " is now " << (nowOccupied ? "OCCUPIED" : "AVAILABLE") << "\n";
    }
};

// ---------- ParkingSpot ----------
class ParkingSpot {
    std::string id;
    SpotType type;
    std::unique_ptr<SpotState> state;
    Vehicle* occupant = nullptr;
    DisplayBoard& display;

public:
    ParkingSpot(std::string id_, SpotType type_, DisplayBoard& disp);

    const std::string& getId() const { return id; }
    SpotType getType() const { return type; }
    bool isAvailable() const { return occupant == nullptr; }
    void setOccupant(Vehicle* v) { occupant = v; }
    Vehicle* getOccupant() const { return occupant; }
    void setState(std::unique_ptr<SpotState> s) { state = std::move(s); }
    std::string currentStateName() const { return state->name(); }

    bool assign(Vehicle* vehicle) { return state->assign(*this, vehicle); }
    void free() { state->free(*this); }

    void notifyDisplay(bool occupied) { display.update(id, occupied); }
};

class AvailableState : public SpotState {
public:
    bool assign(ParkingSpot& spot, Vehicle* vehicle) override;
    std::string name() const override { return "Available"; }
};

class OccupiedState : public SpotState {
public:
    void free(ParkingSpot& spot) override;
    std::string name() const override { return "Occupied"; }
};

ParkingSpot::ParkingSpot(std::string id_, SpotType type_, DisplayBoard& disp)
    : id(std::move(id_)), type(type_), display(disp) {
    state = std::make_unique<AvailableState>();
}

bool AvailableState::assign(ParkingSpot& spot, Vehicle* vehicle) {
    spot.setOccupant(vehicle);
    spot.setState(std::make_unique<OccupiedState>());
    spot.notifyDisplay(true);
    return true;
}

void OccupiedState::free(ParkingSpot& spot) {
    spot.setOccupant(nullptr);
    spot.setState(std::make_unique<AvailableState>());
    spot.notifyDisplay(false);
}

// ---------- Floor ----------
class ParkingFloor {
    std::vector<std::unique_ptr<ParkingSpot>> spots;

public:
    void addSpot(std::unique_ptr<ParkingSpot> spot) { spots.push_back(std::move(spot)); }

    ParkingSpot* findAvailable(SpotType type) {
        for (auto& s : spots) {
            if (s->getType() == type && s->isAvailable()) return s.get();
        }
        return nullptr;
    }
};

// ---------- Allocation Strategy ----------
class SpotAllocationStrategy {
public:
    virtual ~SpotAllocationStrategy() = default;
    virtual ParkingSpot* findSpot(Vehicle* vehicle, std::vector<std::unique_ptr<ParkingFloor>>& floors) = 0;
};

// Best-fit: tries each compatible spot type in the vehicle's preferred order,
// across all floors, before "upgrading" to a larger type.
class BestFitStrategy : public SpotAllocationStrategy {
public:
    ParkingSpot* findSpot(Vehicle* vehicle, std::vector<std::unique_ptr<ParkingFloor>>& floors) override {
        for (SpotType type : vehicle->compatibleSpotTypes()) {
            for (auto& floor : floors) {
                if (auto* spot = floor->findAvailable(type)) return spot;
            }
        }
        return nullptr;
    }
};

// ---------- Ticket ----------
struct Ticket {
    std::string vehiclePlate;
    std::string spotId;
    time_t entryTime;
};

// ---------- Fee Calculator ----------
class FeeCalculator {
public:
    virtual ~FeeCalculator() = default;
    virtual double calculateFee(const Ticket& ticket, time_t exitTime) = 0;
};

class HourlyFeeCalculator : public FeeCalculator {
    double ratePerHour;
public:
    explicit HourlyFeeCalculator(double rate) : ratePerHour(rate) {}
    double calculateFee(const Ticket& ticket, time_t exitTime) override {
        double hours = std::max(1.0, difftime(exitTime, ticket.entryTime) / 3600.0); // minimum 1 hour
        return hours * ratePerHour;
    }
};

// ---------- Payment (Proxy) ----------
class PaymentProcessor {
public:
    bool charge(double amount) {
        std::cout << "Charged $" << amount << " for parking\n";
        return true;
    }
};

// ---------- ParkingLot (Facade) ----------
class ParkingLot {
    std::vector<std::unique_ptr<ParkingFloor>> floors;
    std::unique_ptr<SpotAllocationStrategy> allocationStrategy;
    std::unique_ptr<FeeCalculator> feeCalculator;
    PaymentProcessor payment;
    // maps spotId -> ParkingSpot* would be needed for a full implementation;
    // simplified here by keeping the spot pointer directly on the Ticket via closure/demo scope.

public:
    ParkingLot(std::unique_ptr<SpotAllocationStrategy> alloc, std::unique_ptr<FeeCalculator> fee)
        : allocationStrategy(std::move(alloc)), feeCalculator(std::move(fee)) {}

    void addFloor(std::unique_ptr<ParkingFloor> floor) { floors.push_back(std::move(floor)); }

    // Returns nullptr if no spot available for this vehicle
    std::pair<Ticket, ParkingSpot*> parkVehicle(Vehicle* vehicle) {
        ParkingSpot* spot = allocationStrategy->findSpot(vehicle, floors);
        if (!spot) {
            std::cout << "No available spot for vehicle " << vehicle->licensePlate() << "\n";
            return {Ticket{}, nullptr};
        }
        spot->assign(vehicle);
        Ticket ticket{vehicle->licensePlate(), spot->getId(), time(nullptr)};
        std::cout << "Ticket issued: " << ticket.vehiclePlate << " -> Spot " << ticket.spotId << "\n";
        return {ticket, spot};
    }

    void unparkVehicle(const Ticket& ticket, ParkingSpot* spot) {
        time_t now = time(nullptr);
        double fee = feeCalculator->calculateFee(ticket, now);
        if (payment.charge(fee)) {
            spot->free();
            std::cout << "Vehicle " << ticket.vehiclePlate << " exited, fee $" << fee << "\n";
        }
    }
};

// ---------- Demo ----------
int main() {
    DisplayBoard display;

    auto floor1 = std::make_unique<ParkingFloor>();
    floor1->addSpot(std::make_unique<ParkingSpot>("F1-M1", SpotType::MOTORCYCLE, display));
    floor1->addSpot(std::make_unique<ParkingSpot>("F1-C1", SpotType::COMPACT, display));
    floor1->addSpot(std::make_unique<ParkingSpot>("F1-L1", SpotType::LARGE, display));

    ParkingLot lot(std::make_unique<BestFitStrategy>(), std::make_unique<HourlyFeeCalculator>(2.50));
    lot.addFloor(std::move(floor1));

    Car car("KA-01-1234");
    auto [ticket, spot] = lot.parkVehicle(&car);

    if (spot) {
        lot.unparkVehicle(ticket, spot);
    }

    return 0;
}
```

---

## 10. Trade-offs

- **BestFitStrategy vs NearestSpotStrategy** — best-fit (assign the smallest compatible spot type, preserving larger spots for vehicles that truly need them) is fairer at the lot level but can mean a small vehicle walks farther if the nearest spot happens to be a larger type; nearest-spot minimizes walking distance but can waste large spots on small vehicles when the lot is busy — a real system might combine both (best-fit among the nearest few floors) as a hybrid [[strategy]].
- **In-memory spot lookup vs indexed availability tracking** — the demo's `findAvailable()` does a linear scan per floor; a large multi-floor lot (thousands of spots) would want a maintained per-type-per-floor **counter** (incremented/decremented on assign/free) so availability queries and allocation don't require scanning every spot — an important [[indexing]]-style optimization once scale matters.
- **State pattern for a two-state spot vs a simple boolean `occupied` flag** — Available/Occupied as a boolean is arguably simpler for just two states with no complex transition rules; the State pattern is used here mainly for consistency with the rest of the vault's LLD approach and to make it trivial to extend later (e.g. adding a `ReservedState` for pre-booked parking, or an `OutOfServiceState` for a spot under maintenance) without restructuring.
- **Minimum 1-hour billing vs precise per-minute billing** — the demo's `HourlyFeeCalculator` rounds up to a minimum of 1 hour, a common real-world policy that simplifies pricing at the cost of feeling unfair for very short stays; a per-minute or per-15-minute granularity is a straightforward [[strategy]] swap if the business wants that instead.
- **Single ParkingLot instance vs distributed multi-location system** — this design models one physical lot; a chain operating many lots across a city would need each lot's `ParkingLot` to report availability to a central system (e.g. a mobile app showing "3 lots nearby with space"), introducing the same central-coordination-vs-per-location-autonomy trade-off seen in [[elevator]]'s zoned-dispatch discussion.

---

## 11. Interview Questions

- **How do you prevent two vehicles from being assigned the same spot concurrently at two different entry gates?** The spot's `Available → Occupied` transition must be atomic with respect to concurrent `assign()` calls — in a single-process demo a simple lock suffices, but a real multi-gate, possibly multi-process system needs a database-level conditional update (`UPDATE spots SET occupied=true WHERE id=? AND occupied=false`) or a distributed lock, the same concurrency-correctness concern as [[movie-booking]]'s seat-hold problem.
- **Why allow a vehicle to use a larger spot type when its own is full, but not the reverse?** This is a real-world physical constraint (a bus simply doesn't fit in a compact spot) modeled directly in `Vehicle::compatibleSpotTypes()` — each vehicle type declares an ordered list of spot types it can physically occupy, and the allocation strategy only ever searches within that list, so the constraint is enforced by data rather than scattered conditionals.
- **How would you efficiently answer "how many compact spots are free on floor 3" without scanning every spot?** Maintain a live counter per `(floor, spotType)` pair, incremented on `free()` and decremented on `assign()` (updated via the same [[observer]] notification already used for the `DisplayBoard`) — turns an O(n) scan into an O(1) lookup, essential once a lot has thousands of spots.
- **How would you handle a lost or damaged ticket at exit?** Fall back to matching the vehicle by license plate (captured optically or manually at entry) against the system's active-tickets record instead of relying solely on the physical ticket, and apply a documented "lost ticket" fee/policy as a business rule layered on top of the normal `FeeCalculator`.
- **How would this design change to support reserved/pre-booked parking (e.g. an app lets someone reserve a spot in advance)?** Add a `ReservedState` to the spot's [[state]] machine — a spot can be booked ahead of time (`Available → Reserved`), and only the reserving vehicle's `assign()` call succeeds when they arrive (`Reserved → Occupied`); anyone else attempting to park there is rejected, the same pattern used for reservation-honoring in [[library]]'s book-checkout design.
- **How would you extend the fee model to support dynamic/surge pricing during high-demand periods (e.g. a stadium event)?** Since `FeeCalculator` is already a [[strategy]], swap in (or compose) a demand-aware calculator that factors in current occupancy percentage or time-of-day, without touching `ParkingLot`'s core entry/exit flow — the same reasoning as pricing strategies discussed in [[movie-booking]].

---

## Related Notes

- [[atm]]
- [[elevator]]
- [[food-delivery]]
- [[library]]
- [[movie-booking]]
- [[state]]
- [[strategy]]
- [[observer]]
- [[facade]]
- [[factory-method]]
- [[singleton]]
- [[transactions]]
- [[isolation-levels]]
- [[indexing]]