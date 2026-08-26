# Vending Machine

## 1. Requirements

**Functional**

- Display available products with price and stock count
- Accept coins/cash (and optionally card) as payment, tracking total amount inserted
- Let the user select a product
- Dispense the product if enough money was inserted and it's in stock
- Return change if overpayment occurred (and refuse/refund if change can't be made)
- Support canceling a transaction and refunding inserted money
- Restock products (admin/operator action)
- Handle out-of-stock and insufficient-funds cases gracefully

**Non-Functional**

- The machine must always be in a **well-defined state** — no invalid transitions (e.g. dispensing a product with no money inserted)
- Adding a new payment method (card, mobile pay) shouldn't require rewriting the core transaction flow
- Inventory and cash tracking must stay consistent even if a transaction is interrupted/cancelled mid-way
- Should be straightforward to extend to multiple product slots/rows with independent stock counts

---

## 2. Actors

- **Customer** — selects a product, inserts money, receives product/change
- **Operator/Admin** — restocks products, collects cash, checks inventory

---

## 3. Use Cases

- Customer views available products and prices
- Customer inserts money
- Customer selects a product
- System validates: product exists, in stock, sufficient funds inserted
- System dispenses product and returns change if any
- Customer cancels transaction before selecting → money refunded
- Admin restocks a product slot
- Admin views current inventory and cash collected

---

## 4. Classes

- `Product` — id, name, price
- `Inventory` — maps product slot → `Product` + stock count
- `Coin` / `Money` (enum or class) — represents accepted denominations
- `VendingMachineState` (interface) — represents current machine state; state-specific behavior for `insertMoney()`, `selectProduct()`, `dispense()`, `cancel()`
    - `IdleState`, `HasMoneyState`, `ProductSelectedState`, `DispensingState`
- `VendingMachine` — the context holding current state, inventory, and current transaction's inserted amount; delegates actions to the current `VendingMachineState`
- `PaymentStrategy` (interface, optional) — abstracts payment method (cash vs card)
- `Transaction` — records a completed purchase (product, amount paid, change returned) for audit/logging

---

## 5. Relationships

- `VendingMachine` **has-a** current `VendingMachineState` (state pattern — behavior changes based on state, without a giant conditional)
- `VendingMachine` **has-a** `Inventory` (composition)
- `Inventory` **has-many** `Product` (aggregation, keyed by slot)
- `VendingMachineState` implementations **each hold a reference back to** `VendingMachine` to trigger state transitions
- `VendingMachine` optionally **uses-a** `PaymentStrategy` (strategy pattern) if supporting multiple payment methods

---

## 6. Design Patterns

- **State** — the core pattern here. Machine behavior (`insertMoney`, `selectProduct`, `dispense`, `cancel`) depends entirely on the current state, and each state decides what's legal and what the next state is — avoids a large `if/else`/`switch` on an internal status flag scattered across every method
- **Strategy** — `PaymentStrategy` decouples "how payment is accepted" (cash counting vs card charge) from the core state machine, so new payment types can be added independently
- **Singleton** — a physical vending machine has exactly one instance of itself in a given process; `VendingMachine` is a natural (if debatable) Singleton candidate in a simple embedded-systems context
- **Observer** (optional) — notify an operator/monitoring system when stock runs low or cash reaches a threshold, without the core transaction flow needing to know about monitoring

---

## 7. Class Diagram

```text
┌────────────────────┐        ┌──────────────────────┐
│    VendingMachine     │───────▶│  <<interface>>          │
│ - currentState          │        │  VendingMachineState      │
│ - inventory: Inventory  │        │  + insertMoney(amount)     │
│ - balance: double         │        │  + selectProduct(slot)      │
│ + insertMoney(amt)         │        │  + dispense()                  │
│ + selectProduct(slot)        │        │  + cancel()                      │
│ + dispense()                    │        └──────────┬───────────┘
│ + cancel()                        │                   │ implemented by
│ + setState(state)                   │   ┌───────────────┼────────────────┐
└─────────┬──────────┘   ▼                ▼                 ▼
          │            ┌──────────┐  ┌───────────────┐ ┌──────────────────┐
          │            │IdleState  │  │HasMoneyState    │ │ProductSelectedState│
          │            └──────────┘  └───────────────┘ └──────────────────┘
          ▼
┌────────────────────┐
│      Inventory        │
│ - slots: Map<slot,     │
│     (Product, count)>   │
│ + getProduct(slot)        │
│ + reduceStock(slot)         │
│ + isAvailable(slot)           │
└────────────────────┘
```

---

## 8. Sequence Flow

```text
Customer → VendingMachine.insertMoney(100)
  │  (state: Idle) → balance += 100 → transition to HasMoneyState
  ▼
Customer → VendingMachine.selectProduct(slotA5)
  │  (state: HasMoney)
  ▼
Inventory.isAvailable(slotA5)?
  │  false → error "Out of stock", stay in HasMoneyState (customer can pick another or cancel)
  │  true  ▼
balance >= Product.price?
  │  false → error "Insufficient funds", stay in HasMoneyState
  │  true  ▼
transition to ProductSelectedState
  │
  ▼
VendingMachine.dispense()
  │  (state: ProductSelected)
  ▼
Inventory.reduceStock(slotA5)
change = balance - Product.price
if change > 0 → return change to customer
balance = 0 → transition back to IdleState
```

---

## 9. Code

```cpp
#include <iostream>
#include <unordered_map>
#include <memory>
#include <stdexcept>

struct Product {
    std::string name;
    double price;
};

class Inventory {
    std::unordered_map<std::string, Product> products;
    std::unordered_map<std::string, int> stock;
public:
    void addProduct(const std::string& slot, const Product& p, int count) {
        products[slot] = p;
        stock[slot] = count;
    }

    bool isAvailable(const std::string& slot) const {
        auto it = stock.find(slot);
        return it != stock.end() && it->second > 0;
    }

    Product getProduct(const std::string& slot) const {
        return products.at(slot);
    }

    void reduceStock(const std::string& slot) {
        stock[slot]--;
    }
};

class VendingMachine; // forward declaration

// State pattern — interface for machine states
class VendingMachineState {
public:
    virtual void insertMoney(VendingMachine& machine, double amount) = 0;
    virtual void selectProduct(VendingMachine& machine, const std::string& slot) = 0;
    virtual void dispense(VendingMachine& machine) = 0;
    virtual void cancel(VendingMachine& machine) = 0;
    virtual ~VendingMachineState() = default;
};

class VendingMachine {
    std::unique_ptr<VendingMachineState> currentState;
    Inventory inventory;
    double balance = 0.0;
    std::string selectedSlot;

public:
    explicit VendingMachine(Inventory inv);

    void setState(std::unique_ptr<VendingMachineState> state) { currentState = std::move(state); }
    void insertMoney(double amount) { currentState->insertMoney(*this, amount); }
    void selectProduct(const std::string& slot) { currentState->selectProduct(*this, slot); }
    void dispense() { currentState->dispense(*this); }
    void cancel() { currentState->cancel(*this); }

    double getBalance() const { return balance; }
    void addBalance(double amount) { balance += amount; }
    void resetBalance() { balance = 0; }
    void setSelectedSlot(const std::string& slot) { selectedSlot = slot; }
    std::string getSelectedSlot() const { return selectedSlot; }
    Inventory& getInventory() { return inventory; }
};

// Forward-declare concrete states
class IdleState;
class HasMoneyState;
class ProductSelectedState;

class IdleState : public VendingMachineState {
public:
    void insertMoney(VendingMachine& machine, double amount) override;
    void selectProduct(VendingMachine&, const std::string&) override {
        throw std::logic_error("Insert money before selecting a product");
    }
    void dispense(VendingMachine&) override { throw std::logic_error("No product selected"); }
    void cancel(VendingMachine&) override { /* nothing to cancel */ }
};

class HasMoneyState : public VendingMachineState {
public:
    void insertMoney(VendingMachine& machine, double amount) override {
        machine.addBalance(amount); // allow adding more before selecting
    }
    void selectProduct(VendingMachine& machine, const std::string& slot) override;
    void dispense(VendingMachine&) override { throw std::logic_error("Select a product first"); }
    void cancel(VendingMachine& machine) override;
};

class ProductSelectedState : public VendingMachineState {
public:
    void insertMoney(VendingMachine&, double) override {
        throw std::logic_error("Cannot insert money while dispensing");
    }
    void selectProduct(VendingMachine&, const std::string&) override {
        throw std::logic_error("Product already selected");
    }
    void dispense(VendingMachine& machine) override;
    void cancel(VendingMachine& machine) override;
};

// --- Method definitions (need full VendingMachine visibility) ---
void IdleState::insertMoney(VendingMachine& machine, double amount) {
    machine.addBalance(amount);
    machine.setState(std::make_unique<HasMoneyState>());
}

void HasMoneyState::selectProduct(VendingMachine& machine, const std::string& slot) {
    Inventory& inv = machine.getInventory();
    if (!inv.isAvailable(slot)) throw std::logic_error("Out of stock");
    Product p = inv.getProduct(slot);
    if (machine.getBalance() < p.price) throw std::logic_error("Insufficient funds");

    machine.setSelectedSlot(slot);
    machine.setState(std::make_unique<ProductSelectedState>());
}

void HasMoneyState::cancel(VendingMachine& machine) {
    std::cout << "Refunding: " << machine.getBalance() << "\n";
    machine.resetBalance();
    machine.setState(std::make_unique<IdleState>());
}

void ProductSelectedState::dispense(VendingMachine& machine) {
    Inventory& inv = machine.getInventory();
    std::string slot = machine.getSelectedSlot();
    Product p = inv.getProduct(slot);

    inv.reduceStock(slot);
    double change = machine.getBalance() - p.price;
    std::cout << "Dispensing: " << p.name << ", change: " << change << "\n";

    machine.resetBalance();
    machine.setState(std::make_unique<IdleState>());
}

void ProductSelectedState::cancel(VendingMachine& machine) {
    std::cout << "Refunding: " << machine.getBalance() << "\n";
    machine.resetBalance();
    machine.setState(std::make_unique<IdleState>());
}

VendingMachine::VendingMachine(Inventory inv) : inventory(std::move(inv)) {
    currentState = std::make_unique<IdleState>();
}

int main() {
    Inventory inventory;
    inventory.addProduct("A1", {"Chips", 1.50}, 5);

    VendingMachine machine(inventory);
    machine.insertMoney(2.00);
    machine.selectProduct("A1");
    machine.dispense(); // "Dispensing: Chips, change: 0.5"

    return 0;
}
```

---

## 10. Trade-offs

- **State pattern (class-per-state) vs a single enum + switch statement** — State pattern scales cleanly as behavior per state grows (each state is isolated, Open/Closed for new states) but adds more classes for a machine with genuinely few states; a simple enum+switch can be perfectly reasonable for a small, stable state count
- **Allowing money top-up in `HasMoneyState` vs requiring exact change upfront** — allowing top-up (shown above) is better UX but means `selectProduct` must re-check balance each time; requiring exact change simplifies logic but is unrealistic for a real machine
- **Refusing a purchase if the machine can't make correct change vs dispensing without change** — a real machine must track its own coin/cash inventory (not just product inventory) to know if it _can_ make change; the simplified example skips this and always assumes change is available
- **Single global `VendingMachine` instance (Singleton-ish) vs supporting multiple independent machine instances in the same codebase** — a real deployment (many physical machines managed by one backend/admin system) argues against a hard Singleton; better to keep `VendingMachine` a regular class and let a higher-level `MachineManager` hold multiple instances

---

## 11. Interview Questions

- **Why use the State pattern here instead of a `status` enum with `if/else` checks in every method?** Every method (`insertMoney`, `selectProduct`, `dispense`, `cancel`) would otherwise need its own switch over every possible status, and adding a new state means touching every one of those methods. The State pattern isolates each state's legal transitions into its own class — adding a new state (e.g. a `MaintenanceState`) means adding one new class, not editing four existing ones.
- **How would you track the machine's own cash inventory so it can correctly determine if it can make change?** Add a `CashInventory` (parallel to `Inventory` for products) tracking coin/bill denominations and counts; before completing a dispense, compute if the required change can be made from available denominations (a greedy or DP coin-change check) — if not, refuse the transaction and refund instead of dispensing without correct change.
- **How would you add support for card payments alongside cash?** Introduce a `PaymentStrategy` interface (`CashPayment`, `CardPayment`) that the state machine delegates to for "has payment been received / how much" — the core state transitions (`Idle` → `HasMoney`/`Authorized` → `ProductSelected` → dispense) stay the same; only the payment-collection mechanism changes, which is exactly what Strategy is for.
- **What happens if the machine loses power mid-dispense?** This is where persisting transaction state matters — if `ProductSelectedState`/dispense progress isn't persisted before the physical dispense mechanism runs, a crash could leave inventory decremented without the product actually being delivered (or vice versa). A robust design logs "dispense started" before the physical action and reconciles on restart.
- **How would you extend this to multiple simultaneous customers on one machine (unlikely physically, but as a design exercise)?** A single physical dispenser inherently serializes actions, so the more realistic version of this question is about a **backend system managing many machines** — that's a different problem (each physical machine still has its own single-threaded state machine; the backend layer, not this class, handles concurrency across machines).
- **How is this design open for extension if new product types need special handling (e.g. age-restricted items)?** Add a check in `selectProduct` (or a decorator/strategy around `Product`) that validates additional constraints before allowing the state transition — since `HasMoneyState.selectProduct` is the single choke point for that transition, it's a natural place to plug in extra validation without touching other states.