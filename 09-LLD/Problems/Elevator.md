# Elevator System — Low Level Design

## 1. Requirements

### Functional

- User on a floor can press an **external request** button (Up / Down) to call an elevator
- User inside an elevator can press an **internal request** button for a destination floor
- System dispatches the most appropriate elevator to serve each request (elevator selection/scheduling algorithm)
- Elevator moves through floors, stopping to pick up/drop off according to its scheduled stops
- Doors open at each stop, wait, then close automatically (or on user input)
- Support multiple elevators in a single system (a "elevator bank"), coordinated by a central dispatcher
- Display current floor and direction on each elevator (and optionally at each floor's call panel)
- Handle an elevator going out of service (maintenance mode) — dispatcher must stop assigning it new requests
- Overload protection — elevator shouldn't move if weight limit is exceeded (buzzer/hold doors)

### Non-Functional

- **Efficiency** — minimize average wait time and travel time across all requests; avoid unnecessary direction reversals (ties to [[latency]] and [[throughput]] applied to a physical scheduling problem)
- **Fairness** — no request should be starved indefinitely in favor of always prioritizing "closer" requests
- **Scalability** — the dispatch algorithm should work reasonably whether the building has 2 elevators or 20 ([[scalability]])
- **Extensibility** — should be easy to swap in a different scheduling algorithm (peak-hour optimization, one algorithm for a hospital vs an office tower) without restructuring the rest of the system
- **Reliability** — a request must never be silently dropped; an elevator failure mid-service should not lose track of already-accepted requests it was serving (ties to [[reliability]])
- **Consistency** — the set of pending requests must stay consistent even with concurrent presses across many floors/elevators simultaneously (relevant if requests are handled by concurrent threads/dispatcher instances)

---

## 2. Actors

- **Passenger** — presses external floor buttons (Up/Down) and internal destination buttons
- **Dispatcher / Elevator Control System** — decides which elevator serves which request
- **Technician** — takes an elevator in/out of maintenance mode (secondary actor)
- **Elevator (as a system-internal actor)** — physically moves, opens/closes doors, reports its state back to the dispatcher

---

## 3. Use Cases

- Request an elevator from a floor (external call, with direction)
- Request a destination floor from inside an elevator (internal request)
- Dispatcher assigns the optimal elevator to a new external request
- Elevator services its queue of stops in an efficient order
- Elevator changes direction once it has no more requests in its current direction
- Elevator reports current floor/direction/door-state changes to the dispatcher and display panels
- Take an elevator out of service / bring it back into service
- Handle simultaneous requests from multiple floors across multiple elevators

---

## 4. Classes

|Class|Responsibility|
|---|---|
|**ElevatorSystem**|Top-level facade; owns the dispatcher and the set of elevators — see [[facade]]|
|**Dispatcher**|Central coordinator; receives all requests, decides which elevator handles each one via a pluggable [[strategy]]|
|**SchedulingStrategy (interface)**|Declares `selectElevator(request, elevators)` — different algorithms implement this|
|**NearestElevatorStrategy / SCANStrategy / LookStrategy** (concrete strategies)|Concrete elevator-selection algorithms|
|**Elevator**|Models one physical elevator car: current floor, direction, door state, its own request queue; holds a reference to its current [[state]]|
|**ElevatorState (interface)**|Declares state-dependent behavior (`move()`, `openDoor()`, `closeDoor()`, `addRequest()`)|
|**IdleState / MovingUpState / MovingDownState / DoorOpenState / OutOfServiceState** (concrete states)|Each governs what operations are valid and what transitions follow|
|**Request (abstract)**|Represents one service request; subclasses `ExternalRequest` (floor + direction) and `InternalRequest` (destination floor)|
|**RequestQueue / Direction-sorted queues**|Per-elevator data structure holding pending stops, typically split into "up-stops" and "down-stops" for efficient SCAN-style servicing|
|**Door**|Open/close behavior, safety checks (obstruction sensor)|
|**Display**|Shows current floor + direction, per elevator and per floor panel|
|**ElevatorController (observer target)**|Elevator notifies interested [[observer|

---

## 5. Relationships

- **ElevatorSystem "has-a" Dispatcher, and "has-many" Elevator** — composition; the system owns and coordinates all elevators through one dispatcher
- **Dispatcher "uses-a" SchedulingStrategy** — the Dispatcher delegates the actual elevator-selection decision to a pluggable Strategy object, so the algorithm can be swapped without changing the Dispatcher's own code (Strategy pattern)
- **Elevator "has-a" ElevatorState** — composition; the Elevator delegates its behavior to whichever state is current (State pattern), the same structural idea as the ATM's [[atm|state modeling]]
- **Elevator "is observed by" Display and Dispatcher** — Observer pattern: the Elevator (Subject) notifies its Display and the Dispatcher whenever its floor/direction/door-state changes, without the Elevator needing to know what they do with that information
- **Request "is-a" abstract base**, with `ExternalRequest` and `InternalRequest` as subclasses — inheritance; both carry a target floor, but only `ExternalRequest` carries an intended direction (a person waiting on a floor knows which way they want to go before boarding)
- **Elevator "has-a" RequestQueue** — composition; each elevator maintains its own queue of stops to service, independent of other elevators' queues

---

## 6. Design Patterns

- **[[strategy]]** — the Dispatcher's elevator-selection logic (nearest elevator, SCAN/elevator algorithm, look algorithm) is implemented as interchangeable Strategy objects. This lets the system support different dispatch behavior for different building profiles (e.g. peak-hour "zone" strategies for an office tower vs a simple nearest-elevator strategy for a small building) without touching the Dispatcher's core code.
- **[[state]]** — each Elevator's behavior is governed by its current state (`Idle`, `MovingUp`, `MovingDown`, `DoorOpen`, `OutOfService`). This avoids scattering `if direction == UP ... else if direction == DOWN ...` conditionals through every method, and cleanly encodes which operations are valid in which state (e.g. you can't `addRequest()` to an `OutOfServiceState` elevator).
- **[[observer]]** — the Elevator (Subject) notifies registered Observers (floor Displays, the Dispatcher, possibly a logging/monitoring service) whenever its floor, direction, or door state changes — decoupling the Elevator's core logic from everything that needs to react to its state changes.
- **[[factory-method]]** — a `RequestFactory` can create the correct `Request` subclass (`ExternalRequest` vs `InternalRequest`) based on which button was pressed (a floor panel Up/Down button vs an in-car destination button), keeping request-construction logic out of the button-handling code.
- **[[singleton]]** — the `ElevatorSystem` itself (or its Dispatcher) is a natural single-instance-per-building component; as with the ATM's `AuditLogger`, this is best implemented via dependency injection as a singleton-scoped service in real code rather than a classic `getInstance()` (see [[singleton]] trade-offs).
- **[[command]]** (optional extension) — each button press (floor call, destination select) can be modeled as a Command object queued for the Dispatcher, which is a natural fit if requests need to be logged, replayed for debugging, or processed asynchronously through a queue rather than handled synchronously inline.

---

## 7. Class Diagram

```text
                    ┌────────────────────┐
                    │   ElevatorSystem     │  (Facade)
                    │ - dispatcher          │
                    │ - elevators: List      │
                    │ + requestElevator()    │
                    │ + selectFloor(id, floor)│
                    └─────────┬──────────┘
                              │ owns
                              ▼
                    ┌────────────────────┐
                    │     Dispatcher       │
                    │ - strategy: SchedulingStrategy│
                    │ + handleRequest(req)  │
                    └─────────┬──────────┘
                              │ uses
                              ▼
                    ┌────────────────────────┐
                    │ SchedulingStrategy (interface)│
                    │ + selectElevator(req, elevators)│
                    └─────────┬──────────────┘
              ┌───────────────┼────────────────┐
              ▼                ▼                 ▼
    ┌──────────────┐ ┌──────────────┐  ┌──────────────┐
    │NearestElevator│ │  SCANStrategy │  │ LookStrategy  │
    │  Strategy      │ │                │  │                │
    └──────────────┘ └──────────────┘  └──────────────┘

┌────────────────────┐        ┌──────────────────────┐
│      Elevator        │───────▶│ ElevatorState (interface)│
│ - id                  │        │ + move()                 │
│ - currentFloor         │        │ + openDoor()               │
│ - direction            │        │ + closeDoor()               │
│ - requestQueue         │        │ + addRequest(floor)         │
│ - state                │        └─────────┬─────────────┘
│ - observers: List       │                  │ implemented by
│ + addRequest(req)        │        ┌────────┼─────────┬──────────────┐
│ + notifyObservers()       │        ▼         ▼          ▼              ▼
└──────────┬─────────┘  ┌──────────┐┌────────────┐┌────────────┐┌───────────────┐
           │ notifies    │IdleState ││MovingUpState││MovingDownState││OutOfServiceState│
           ▼             └──────────┘└────────────┘└────────────┘└───────────────┘
 ┌──────────────────┐
 │ Observer (interface)│
 │  + update(elevator)   │
 └─────────┬─────────┘
           │ implemented by
   ┌───────┴────────┐
   ▼                  ▼
┌──────────┐  ┌──────────────┐
│ Display   │  │ Dispatcher    │ (also observes, to update its
└──────────┘  │ (as observer)  │  internal view of elevator states)
               └──────────────┘

┌────────────────┐
│ Request (abstract)│
│ # floor            │
└────────┬───────┘
         │ extended by
   ┌─────┴──────┐
   ▼              ▼
┌──────────────┐┌──────────────┐
│ExternalRequest ││InternalRequest│
│ + direction     ││                │
└──────────────┘└──────────────┘
```

---

## 8. Sequence Flow

```text
External Request — Passenger on Floor 3 presses "Up"

Passenger      ElevatorSystem     Dispatcher        SchedulingStrategy   Elevator (chosen)   Display
   │ press "Up" on floor 3│                │                   │                    │            │
   │──────────────────────▶│                │                   │                    │            │
   │                        │──requestElevator(floor=3, dir=UP)─▶│                   │            │
   │                        │                │──selectElevator(req, elevators)───────▶│            │
   │                        │                │                   │ (evaluates each   │            │
   │                        │                │                   │  elevator's cost: │            │
   │                        │                │                   │  distance, direction,          │
   │                        │                │                   │  current load)     │            │
   │                        │                │◀── Elevator #2 ───│                   │            │
   │                        │                │──addRequest(Elevator #2, floor=3)─────────────────▶│
   │                        │                │                   │                    │ (queues stop;
   │                        │                │                   │                    │  if Idle, transitions
   │                        │                │                   │                    │  to MovingUp/DownState)
   │                        │                │                   │                    │──notifyObservers()──▶│
   │                        │                │                   │                    │            │ (updates floor
   │                        │                │                   │                    │            │  indicator: "Elevator
   │                        │                │                   │                    │            │  #2 en route")
   │                        │                │                   │                    │            │
   │ ... elevator arrives at floor 3 ...      │                   │                    │            │
   │                        │                │                   │                    │──openDoor()─┼──────▶│
   │◀── doors open, board ──────────────────────────────────────────────────────────────────────────│
   │ press destination "7"  │                │                   │                    │            │
   │──────────────────────────────────────────────────────────────▶│                    │            │
   │                        │                │                   │──addRequest(InternalRequest(7))──▶│
   │                        │                │                   │                    │ (queued; elevator
   │                        │                │                   │                    │  continues serving
   │                        │                │                   │                    │  stops in direction
   │                        │                │                   │                    │  order until floor 7)
```

---

## 9. Code

```cpp
#include <iostream>
#include <vector>
#include <set>
#include <memory>
#include <algorithm>
#include <limits>

enum class Direction { UP, DOWN, IDLE };

// ---------- Request hierarchy ----------
struct Request {
    int floor;
    virtual ~Request() = default;
};

struct ExternalRequest : Request {
    Direction direction;
    ExternalRequest(int f, Direction d) { floor = f; direction = d; }
};

struct InternalRequest : Request {
    explicit InternalRequest(int f) { floor = f; }
};

// ---------- Observer interface ----------
class Elevator; // fwd decl

class ElevatorObserver {
public:
    virtual ~ElevatorObserver() = default;
    virtual void update(const Elevator& elevator) = 0;
};

class Display : public ElevatorObserver {
public:
    void update(const Elevator& elevator) override;
};

// ---------- Elevator States ----------
class ElevatorState {
public:
    virtual ~ElevatorState() = default;
    virtual void step(Elevator& elevator) = 0; // one simulation tick
    virtual std::string name() const = 0;
};

// ---------- Elevator (Context for State, Subject for Observer) ----------
class Elevator {
    int id;
    int currentFloor = 0;
    Direction direction = Direction::IDLE;
    bool doorOpen = false;
    bool outOfService = false;
    std::set<int> upStops, downStops; // split queues for SCAN-style servicing
    std::unique_ptr<ElevatorState> state;
    std::vector<ElevatorObserver*> observers;

public:
    explicit Elevator(int id);

    int getId() const { return id; }
    int getCurrentFloor() const { return currentFloor; }
    Direction getDirection() const { return direction; }
    bool isDoorOpen() const { return doorOpen; }
    bool isOutOfService() const { return outOfService; }
    bool hasPendingRequests() const { return !upStops.empty() || !downStops.empty(); }

    void setState(std::unique_ptr<ElevatorState> newState) { state = std::move(newState); }
    void setDirection(Direction d) { direction = d; }
    void setDoorOpen(bool open) { doorOpen = open; }
    void setFloor(int f) { currentFloor = f; }

    void addObserver(ElevatorObserver* o) { observers.push_back(o); }
    void notifyObservers() { for (auto* o : observers) o->update(*this); }

    void addRequest(const Request& req) {
        if (outOfService) { std::cout << "Elevator " << id << " is out of service\n"; return; }
        if (req.floor > currentFloor) upStops.insert(req.floor);
        else if (req.floor < currentFloor) downStops.insert(req.floor);
        // if req.floor == currentFloor, treat as immediate stop (simplified: ignored here)
        notifyObservers();
    }

    bool nextUpStop(int& out) const { if (upStops.empty()) return false; out = *upStops.begin(); return true; }
    bool nextDownStop(int& out) const { if (downStops.empty()) return false; out = *downStops.rbegin(); return true; }
    void removeUpStop(int floor) { upStops.erase(floor); }
    void removeDownStop(int floor) { downStops.erase(floor); }

    // "cost" estimate used by the scheduling strategy — simplistic distance-based cost
    int costFor(const ExternalRequest& req) const {
        if (outOfService) return std::numeric_limits<int>::max();
        int distance = std::abs(currentFloor - req.floor);
        bool movingTowardAndSameDirection =
            (direction == req.direction) &&
            ((req.direction == Direction::UP && req.floor >= currentFloor) ||
             (req.direction == Direction::DOWN && req.floor <= currentFloor));
        if (direction == Direction::IDLE) return distance;
        if (movingTowardAndSameDirection) return distance;
        return distance + 1000; // heavy penalty for wrong-direction / already-passed requests
    }

    void step(); // delegates to state
};

void Display::update(const Elevator& elevator) {
    std::cout << "[Display] Elevator " << elevator.getId()
              << " at floor " << elevator.getCurrentFloor()
              << (elevator.isDoorOpen() ? " (door open)" : "") << "\n";
}

// ---------- Concrete States ----------
class IdleState : public ElevatorState {
public:
    void step(Elevator& elevator) override {
        int target;
        if (elevator.nextUpStop(target)) {
            elevator.setDirection(Direction::UP);
        } else if (elevator.nextDownStop(target)) {
            elevator.setDirection(Direction::DOWN);
        } else {
            return; // nothing to do, remain idle
        }
        elevator.setState(std::make_unique<class MovingState>()); // see below (forward pattern simplified)
    }
    std::string name() const override { return "Idle"; }
};

class MovingState : public ElevatorState {
public:
    void step(Elevator& elevator) override {
        int target;
        Direction dir = elevator.getDirection();
        bool hasTarget = (dir == Direction::UP) ? elevator.nextUpStop(target)
                                                 : elevator.nextDownStop(target);
        if (!hasTarget) {
            elevator.setDirection(Direction::IDLE);
            elevator.setState(std::make_unique<IdleState>());
            return;
        }
        int floor = elevator.getCurrentFloor();
        if (floor == target) {
            elevator.setDoorOpen(true);
            if (dir == Direction::UP) elevator.removeUpStop(target);
            else elevator.removeDownStop(target);
            elevator.notifyObservers();
            elevator.setDoorOpen(false); // simplified: door closes immediately next tick
        } else {
            elevator.setFloor(floor + (dir == Direction::UP ? 1 : -1));
            elevator.notifyObservers();
        }
    }
    std::string name() const override { return "Moving"; }
};

Elevator::Elevator(int id_) : id(id_) { state = std::make_unique<IdleState>(); }
void Elevator::step() { state->step(*this); }

// ---------- Scheduling Strategy ----------
class SchedulingStrategy {
public:
    virtual ~SchedulingStrategy() = default;
    virtual Elevator* selectElevator(const ExternalRequest& req, std::vector<std::unique_ptr<Elevator>>& elevators) = 0;
};

class NearestElevatorStrategy : public SchedulingStrategy {
public:
    Elevator* selectElevator(const ExternalRequest& req, std::vector<std::unique_ptr<Elevator>>& elevators) override {
        Elevator* best = nullptr;
        int bestCost = std::numeric_limits<int>::max();
        for (auto& e : elevators) {
            int cost = e->costFor(req);
            if (cost < bestCost) { bestCost = cost; best = e.get(); }
        }
        return best;
    }
};

// ---------- Dispatcher ----------
class Dispatcher {
    std::unique_ptr<SchedulingStrategy> strategy;
    std::vector<std::unique_ptr<Elevator>>& elevators;

public:
    Dispatcher(std::unique_ptr<SchedulingStrategy> strat, std::vector<std::unique_ptr<Elevator>>& elevs)
        : strategy(std::move(strat)), elevators(elevs) {}

    void handleExternalRequest(const ExternalRequest& req) {
        Elevator* chosen = strategy->selectElevator(req, elevators);
        if (!chosen) { std::cout << "No elevator available\n"; return; }
        std::cout << "Dispatcher assigns Elevator " << chosen->getId() << " to floor " << req.floor << "\n";
        chosen->addRequest(req);
    }
};

// ---------- ElevatorSystem (Facade) ----------
class ElevatorSystem {
    std::vector<std::unique_ptr<Elevator>> elevators;
    Dispatcher dispatcher;
    Display display;

public:
    ElevatorSystem(int numElevators)
        : dispatcher(std::make_unique<NearestElevatorStrategy>(), elevators) {
        for (int i = 0; i < numElevators; ++i) {
            auto e = std::make_unique<Elevator>(i);
            e->addObserver(&display);
            elevators.push_back(std::move(e));
        }
    }

    void requestElevator(int floor, Direction dir) {
        dispatcher.handleExternalRequest(ExternalRequest(floor, dir));
    }

    void selectFloor(int elevatorId, int floor) {
        if (elevatorId >= 0 && elevatorId < (int)elevators.size()) {
            elevators[elevatorId]->addRequest(InternalRequest(floor));
        }
    }

    void tick() { // advances simulation by one step for every elevator
        for (auto& e : elevators) e->step();
    }
};

// ---------- Demo ----------
int main() {
    ElevatorSystem system(3);

    system.requestElevator(5, Direction::UP);   // passenger on floor 5 wants to go up
    system.selectFloor(0, 9);                    // assume Elevator 0 was chosen, wants floor 9

    for (int i = 0; i < 15; ++i) {
        system.tick();
    }

    return 0;
}
```

---

## 10. Trade-offs

- **NearestElevatorStrategy vs SCAN/LOOK algorithms** — the simple nearest-elevator strategy shown minimizes wait time for the _current_ request but can cause unnecessary direction reversals and worse aggregate throughput under heavy load; SCAN (elevator continues in one direction, servicing all stops, before reversing) generally gives better average performance for many simultaneous requests, at the cost of slightly higher wait time for some individual requests — a classic latency-vs-throughput trade-off (see [[latency]], [[throughput]]).
- **Split up/down request queues per elevator** — using two sorted sets (`upStops`, `downStops`) makes SCAN-style servicing straightforward (always pick the nearest stop in the current direction) but requires care when a request arrives for a floor behind the elevator's current direction — it must wait in the _other_ queue until the elevator reverses, which can be a source of subtle bugs if not tested carefully.
- **Central Dispatcher as a single coordinator** — simplifies the system, but is a potential bottleneck/single point of failure for a very large building with many elevators and extremely high request volume; a distributed or partitioned dispatch (e.g. elevators grouped into banks per zone/floor range) could reduce this at the cost of more complex coordination (ties to [[scalability]]).
- **State pattern per elevator vs a global system-wide state machine** — modeling state per-elevator (as done here) keeps each elevator independently reasoned about, but means the Dispatcher must query each elevator's state/queue rather than relying on one global view — a reasonable trade favoring elevator independence and simpler per-unit testing.
- **Synchronous `tick()`-based simulation vs event-driven** — the demo uses a simple discrete tick loop for clarity; a real system would be event-driven (elevator hardware reports floor-sensor triggers as they occur), better modeled with the same [[observer]]-based notification already used for Display updates.

---

## 11. Interview Questions

- **How would you design the scheduling algorithm to minimize average wait time?** Start with a cost function per elevator per request (distance, current direction alignment, current load), as shown in `costFor()`; for real systems, SCAN or LOOK algorithms (elevator services all stops in its current direction before reversing) generally outperform naive nearest-elevator assignment under load, because they reduce wasted reversals — implement this as a swappable [[strategy]] so it can be tuned/replaced without touching the Dispatcher.
- **How do you prevent starvation — a request on a floor that never gets served because "nearer" requests keep winning?** Track request wait time and factor it into the cost function (e.g. add a penalty that grows the longer a request has been waiting), or use an algorithm like SCAN that guarantees every pending stop in the current direction is eventually serviced before reversal, rather than always chasing the globally "nearest" request.
- **How would you handle an elevator breaking down mid-route with passengers already inside, who had pending destination requests?** The Elevator should transition to an `OutOfServiceState`; the Dispatcher must be notified (via [[observer]]) so it stops assigning new external requests to that elevator, and ideally the system flags the in-progress internal requests so they can be manually reassigned/communicated to trapped passengers (a case where pure software design meets a hardware/safety escalation path).
- **Why use the State pattern here instead of just an enum `Direction` field with conditionals?** As the elevator gains more distinct behavior beyond just direction (e.g. `DoorOpenState`, `OutOfServiceState`, each disallowing different operations), a plain enum + conditionals scattered across methods gets error-prone (e.g. forgetting to block `addRequest()` while doors are open or the unit is out of service). State pattern centralizes each state's valid operations, the same reasoning applied in the [[atm|ATM design]].
- **How would you extend this design for a very tall building with "zoned" elevators (e.g. some elevators only serve floors 1-20, others 21-40)?** Partition elevators into zone-specific groups at the Dispatcher level (or introduce multiple Dispatchers, one per zone), and route each request first to the zone matching its floor before applying the normal elevator-selection strategy within that zone — reduces the search space per request and can improve overall throughput at scale (ties to [[scalability]]'s sharding-style reasoning, applied to a physical system instead of data).
- **How would concurrent requests from many floors, arriving at nearly the same time, be handled safely?** The Dispatcher's request-handling and each Elevator's queue mutation need to be protected against races if requests are processed on separate threads (e.g. one thread per floor call) — this is conceptually the same concurrency-safety concern discussed in [[transactions]]: either serialize all dispatch decisions through a single-threaded Dispatcher (simplest), or use fine-grained locking per elevator's request queue if the Dispatcher itself is parallelized.

---

## Related Notes

- [[atm]]
- [[state]]
- [[strategy]]
- [[observer]]
- [[factory-method]]
- [[facade]]
- [[singleton]]
- [[scalability]]
- [[latency]]
- [[throughput]]
- [[reliability]]