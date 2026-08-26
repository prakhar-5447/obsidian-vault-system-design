# ATM — Low Level Design

## 1. Requirements

### Functional

- User can authenticate using a card (or card number) and a PIN
- User can check account balance
- User can withdraw cash, in valid denominations, up to their account balance and any withdrawal limit
- User can deposit cash
- User can change their PIN
- System dispenses the correct combination of available currency notes/denominations for a withdrawal
- System prints/displays a transaction receipt (or on-screen summary) for each transaction
- Card is retained/ejected appropriately (e.g. ejected after transaction, retained after N failed PIN attempts)
- Transaction is fully reversible/no-op if any step fails partway (e.g. cash cannot be dispensed) — no money should be debited without cash dispensed, or vice versa

### Non-Functional

- **Consistency** — an account's balance must never be left in an inconsistent state, even if the machine loses power or crashes mid-withdrawal (ties to [[acid]], [[transactions]])
- **Concurrency safety** — the same account being accessed from two ATMs (or an ATM + online banking) simultaneously must not allow double-withdrawal beyond the balance
- **Availability** — a single ATM's hardware fault (e.g. out of cash) shouldn't corrupt state or crash the whole session; it should fail gracefully
- **Security** — PIN must never be stored/logged in plaintext; limited retry attempts before card lock; encrypted communication with the bank server
- **Extensibility** — new transaction types (mobile recharge, bill pay, mini-statement) should be addable without restructuring the core system
- **Auditability** — every transaction must be logged for reconciliation and dispute resolution

---

## 2. Actors

- **Customer** — the human interacting with the ATM to perform transactions
- **Bank Server** — the external system of record for account balances; the ATM is a client to it, never the source of truth
- **Technician / Maintenance Operator** — restocks cash, resolves hardware faults (out of scope for core transaction flow, but relevant to state modeling)
- **ATM Hardware components** (modeled as system-internal actors from the software's point of view) — Card Reader, Keypad, Cash Dispenser, Screen/Display, Printer

---

## 3. Use Cases

- Authenticate (insert card + enter PIN)
- Check balance
- Withdraw cash
- Deposit cash
- Change PIN
- Print/display receipt
- Handle insufficient funds (in account, or in the machine's cash inventory)
- Handle authentication failure / lock card after repeated failed PIN attempts
- Handle transaction failure mid-flow (network drop, dispenser jam) and roll back safely
- Eject / retain card at end of session

---

## 4. Classes

|Class|Responsibility|
|---|---|
|**ATM**|Top-level coordinator; holds current [[state]] and delegates operations to it; owns references to hardware components|
|**ATMState (interface)**|Declares state-dependent operations (`insertCard()`, `enterPin()`, `selectTransaction()`, `ejectCard()`) — see [[state]] pattern|
|**IdleState / HasCardState / AuthenticatedState / TransactionState** (concrete states)|Each implements only the operations valid in that state, and triggers transitions|
|**Card**|Holds card number, associated account reference; inserted/read by CardReader|
|**Account**|Balance, account number, owner; source-of-truth calls go through BankServer, this is a local cached representation|
|**CardReader**|Reads card data, validates card is not expired/blocked|
|**Keypad**|Captures PIN and menu input|
|**CashDispenser**|Tracks available denominations/counts; computes and dispenses the note combination for a withdrawal amount|
|**Screen**|Displays prompts/messages/menus to the customer|
|**Printer**|Prints a transaction receipt|
|**Transaction (abstract)**|Represents one transaction attempt; subclasses: `WithdrawTransaction`, `DepositTransaction`, `BalanceInquiryTransaction`, `ChangePinTransaction`|
|**TransactionFactory**|Creates the correct concrete `Transaction` subclass based on user selection — see [[factory-method]]|
|**BankServer (interface/proxy)**|ATM's gateway to the actual bank backend — validates PIN, checks/updates balance; modeled as a [[proxy]] since the real bank server lives remotely|
|**AuditLogger**|Records every transaction attempt (success or failure) for reconciliation|

---

## 5. Relationships

- **ATM "has-a" ATMState** — composition; the ATM delegates behavior to whichever state is current (State pattern)
- **ATM "has-a" CardReader, Keypad, CashDispenser, Screen, Printer** — composition; these are permanent hardware components of one ATM instance
- **Transaction "is-a" abstract base**, with `WithdrawTransaction`, `DepositTransaction`, etc. as concrete subclasses — inheritance
- **TransactionFactory "creates" Transaction** — Factory Method: the ATM asks the factory for a Transaction based on the user's menu choice, without knowing the concrete subclass
- **BankServer "is used by" ATM and by Transaction** — Transaction delegates the actual balance check/update to BankServer; ATM never touches account balances directly, preserving the invariant that BankServer is the single source of truth
- **Card "has-a" Account reference** — association, resolved via BankServer lookup at authentication time, not stored locally in plaintext on the ATM
- **CashDispenser "used by" WithdrawTransaction** — association; the transaction asks the dispenser to dispense a specific amount, dispenser handles denomination breakdown

---

## 6. Design Patterns

- **[[state]]** — the ATM's behavior (which operations are valid) changes based on its current state: `Idle → HasCard → Authenticated → InTransaction → Idle`. Using State avoids a tangle of `if currentState == X` checks scattered across every method, and each state class cleanly owns its own valid transitions.
- **[[factory-method]]** — `TransactionFactory` creates the correct `Transaction` subclass (`WithdrawTransaction`, `DepositTransaction`, etc.) based on the menu option selected, so the ATM's core loop is written only against the abstract `Transaction` interface and doesn't need a growing conditional every time a new transaction type is added.
- **[[strategy]]** — the `CashDispenser`'s denomination-selection algorithm (which notes to dispense for a given amount) can be swapped as a Strategy, e.g. a "minimize note count" strategy vs a "preserve rare denominations" strategy, without changing the dispenser's core dispensing logic.
- **[[proxy]]** — `BankServer` as seen by the ATM is a **Remote Proxy**: the ATM calls what looks like a local interface (`checkBalance()`, `debit()`), but the Proxy actually handles the network call to the real, remote bank backend, hiding that complexity from the rest of the ATM's code.
- **[[singleton]]** — the `AuditLogger` (and potentially the `CashDispenser`'s inventory tracker) is a natural single-instance-per-ATM component; in production code this would ideally be dependency-injected as a singleton-scoped service rather than a classic `getInstance()` singleton, to keep it testable (see [[singleton]] trade-offs).
- **[[command]]** (optional extension) — each transaction step could be modeled as a Command object, enabling a transaction log that supports replay for audit/reconciliation, or a rollback ("undo") path if a step fails mid-transaction.

---

## 7. Class Diagram

```text
                        ┌────────────────────┐
                        │        ATM          │
                        │ - state: ATMState    │
                        │ - cardReader          │
                        │ - keypad              │
                        │ - cashDispenser       │
                        │ - screen              │
                        │ - printer             │
                        │ - bankServer          │
                        │ + insertCard()         │
                        │ + enterPin()           │
                        │ + selectTransaction()  │
                        │ + ejectCard()          │
                        └─────────┬──────────┘
                                  │ delegates to
                                  ▼
                        ┌────────────────────┐
                        │  ATMState (interface)│
                        │ + insertCard(atm)      │
                        │ + enterPin(atm, pin)   │
                        │ + selectTransaction()  │
                        │ + ejectCard(atm)       │
                        └─────────┬──────────┘
              ┌───────────┬───────┴────────┬───────────────┐
              ▼           ▼                ▼               ▼
      ┌──────────┐┌───────────────┐┌──────────────────┐┌──────────┐
      │IdleState  ││ HasCardState   ││AuthenticatedState ││InTransaction│
      └──────────┘└───────────────┘└──────────────────┘│  State      │
                                                          └──────────┘

┌────────────────┐        ┌──────────────────────┐
│ TransactionFactory│──────▶│  Transaction (abstract) │
│ + create(type)     │      │ # account               │
└────────────────┘        │ # bankServer            │
                            │ + execute(): Result      │
                            └──────────┬───────────┘
                                       │ extended by
                 ┌──────────┬─────────┼─────────┬──────────────┐
                 ▼           ▼                    ▼              ▼
        ┌────────────┐┌────────────┐   ┌──────────────────┐┌────────────┐
        │WithdrawTxn   ││DepositTxn   │   │BalanceInquiryTxn  ││ChangePinTxn│
        │+ execute()    ││+ execute()   │   │+ execute()          ││+ execute()  │
        └────────────┘└────────────┘   └──────────────────┘└────────────┘

┌────────────────┐       ┌────────────────────┐
│  CashDispenser   │       │  BankServer (proxy)  │
│ - inventory[denom]│       │ + authenticate(card,pin)│
│ + dispense(amount)│       │ + getBalance(acct)     │
│ + canDispense(amt)│       │ + debit(acct, amt)      │
└────────────────┘       │ + credit(acct, amt)     │
                           └────────────────────┘
```

---

## 8. Sequence Flow

```text
Withdraw Cash — Happy Path

Customer          ATM (IdleState)      CardReader     ATMState(chain)     BankServer (proxy)   CashDispenser   AuditLogger
   │ insert card       │                    │                │                    │                 │              │
   │──────────────────▶│                    │                │                    │                 │              │
   │                    │──readCard()───────▶│                │                    │                 │              │
   │                    │                    │──card data─────▶│ (transition to     │                 │              │
   │                    │                    │                │  HasCardState)     │                 │              │
   │  enter PIN         │                    │                │                    │                 │              │
   │───────────────────▶│                    │                │                    │                 │              │
   │                    │──enterPin(pin)─────┼───────────────▶│──authenticate()───▶│                 │              │
   │                    │                    │                │◀──── OK ───────────│                 │              │
   │                    │                    │                │ (transition to     │                 │              │
   │                    │                    │                │  AuthenticatedState)│                │              │
   │  select "Withdraw", enter $200          │                │                    │                 │              │
   │───────────────────▶│                    │                │                    │                 │              │
   │                    │──TransactionFactory.create(WITHDRAW, 200)                 │                 │              │
   │                    │        │           │                │                    │                 │              │
   │                    │        ▼           │                │                    │                 │              │
   │                    │  WithdrawTransaction.execute()       │                    │                 │              │
   │                    │                    │                │──getBalance(acct)─▶│                 │              │
   │                    │                    │                │◀──── $500 ─────────│                 │              │
   │                    │                    │                │  (balance sufficient)                │              │
   │                    │                    │                │──canDispense($200)─┼────────────────▶│              │
   │                    │                    │                │◀──── yes ──────────┼─────────────────│              │
   │                    │                    │                │──debit(acct, $200)▶│                 │              │
   │                    │                    │                │◀──── OK ───────────│                 │              │
   │                    │                    │                │──dispense($200)────┼────────────────▶│              │
   │                    │                    │                │                    │  (physically dispenses notes)  │
   │                    │                    │                │──log(txn, success)─┼─────────────────┼─────────────▶│
   │◀── cash + receipt ─│                    │                │                    │                 │              │
   │  eject card         │                    │                │ (transition back   │                 │              │
   │◀───────────────────│                    │                │  to IdleState)     │                 │              │

Failure branch: if debit() succeeds but dispense() then fails (jam/out-of-cash),
WithdrawTransaction must issue a compensating credit(acct, $200) back to BankServer
before reporting failure to the customer — never leave the account debited with no
cash dispensed (see [[acid]] atomicity, [[idempotency]] for safe retry semantics).
```

---

## 9. Code

```cpp
#include <iostream>
#include <string>
#include <map>
#include <memory>
#include <stdexcept>

// ---------- Forward declarations ----------
class ATM;

// ---------- Bank Server (Remote Proxy) ----------
class BankServer {
public:
    virtual ~BankServer() = default;
    virtual bool authenticate(const std::string& cardNumber, const std::string& pin) = 0;
    virtual double getBalance(const std::string& accountId) = 0;
    virtual bool debit(const std::string& accountId, double amount) = 0;
    virtual bool credit(const std::string& accountId, double amount) = 0;
};

// Simplified in-memory stand-in for the real remote bank backend
class BankServerProxy : public BankServer {
    std::map<std::string, double> balances{{"acct-1", 500.0}};
    std::map<std::string, std::string> pins{{"card-1", "1234"}};
    std::map<std::string, std::string> cardToAccount{{"card-1", "acct-1"}};

public:
    bool authenticate(const std::string& cardNumber, const std::string& pin) override {
        auto it = pins.find(cardNumber);
        return it != pins.end() && it->second == pin;
    }

    double getBalance(const std::string& accountId) override {
        return balances.at(accountId);
    }

    bool debit(const std::string& accountId, double amount) override {
        if (balances[accountId] < amount) return false;
        balances[accountId] -= amount;
        return true;
    }

    bool credit(const std::string& accountId, double amount) override {
        balances[accountId] += amount;
        return true;
    }

    std::string accountFor(const std::string& cardNumber) {
        return cardToAccount.at(cardNumber);
    }
};

// ---------- Cash Dispenser (Strategy-ready denomination logic) ----------
class CashDispenser {
    std::map<int, int> inventory{{2000, 5}, {500, 20}, {100, 50}}; // denom -> count

public:
    bool canDispense(double amount) const {
        // simplified: assumes greedy denomination breakdown is always exact for this demo
        return amount > 0 && static_cast<int>(amount) % 100 == 0;
    }

    bool dispense(double amount) {
        int remaining = static_cast<int>(amount);
        std::map<int, int> toDispense;
        for (auto& [denom, count] : inventory) {
            // NOTE: map iterates ascending; a real implementation would iterate
            // descending denominations first for a minimal-notes greedy strategy
        }
        for (auto it = inventory.rbegin(); it != inventory.rend(); ++it) {
            int denom = it->first;
            int& available = it->second;
            int use = std::min(available, remaining / denom);
            if (use > 0) {
                toDispense[denom] = use;
                remaining -= use * denom;
            }
        }
        if (remaining != 0) return false; // couldn't make exact change
        for (auto& [denom, count] : toDispense) {
            inventory[denom] -= count;
            std::cout << "Dispensing " << count << " x $" << denom << " notes\n";
        }
        return true;
    }
};

// ---------- Audit Logger (Singleton-scoped in real systems) ----------
class AuditLogger {
public:
    void log(const std::string& txnType, bool success) {
        std::cout << "[AUDIT] " << txnType << " -> " << (success ? "SUCCESS" : "FAILURE") << "\n";
    }
};

// ---------- Transaction hierarchy ----------
class Transaction {
protected:
    std::string accountId;
    BankServer& bankServer;
    CashDispenser& dispenser;
    AuditLogger& logger;

public:
    Transaction(std::string accountId, BankServer& bank, CashDispenser& disp, AuditLogger& log)
        : accountId(std::move(accountId)), bankServer(bank), dispenser(disp), logger(log) {}

    virtual ~Transaction() = default;
    virtual bool execute() = 0;
};

class WithdrawTransaction : public Transaction {
    double amount;

public:
    WithdrawTransaction(std::string acct, double amt, BankServer& bank, CashDispenser& disp, AuditLogger& log)
        : Transaction(std::move(acct), bank, disp, log), amount(amt) {}

    bool execute() override {
        if (bankServer.getBalance(accountId) < amount) {
            logger.log("WITHDRAW", false);
            return false;
        }
        if (!dispenser.canDispense(amount)) {
            logger.log("WITHDRAW", false);
            return false;
        }
        if (!bankServer.debit(accountId, amount)) {
            logger.log("WITHDRAW", false);
            return false;
        }
        if (!dispenser.dispense(amount)) {
            // compensating action — dispenser failed AFTER debit succeeded
            bankServer.credit(accountId, amount);
            logger.log("WITHDRAW", false);
            return false;
        }
        logger.log("WITHDRAW", true);
        return true;
    }
};

class BalanceInquiryTransaction : public Transaction {
public:
    using Transaction::Transaction;

    bool execute() override {
        double balance = bankServer.getBalance(accountId);
        std::cout << "Balance: $" << balance << "\n";
        logger.log("BALANCE_INQUIRY", true);
        return true;
    }
};

// ---------- Transaction Factory ----------
enum class TransactionType { WITHDRAW, BALANCE_INQUIRY };

class TransactionFactory {
public:
    static std::unique_ptr<Transaction> create(
        TransactionType type, const std::string& accountId, double amount,
        BankServer& bank, CashDispenser& disp, AuditLogger& log) {
        switch (type) {
            case TransactionType::WITHDRAW:
                return std::make_unique<WithdrawTransaction>(accountId, amount, bank, disp, log);
            case TransactionType::BALANCE_INQUIRY:
                return std::make_unique<BalanceInquiryTransaction>(accountId, bank, disp, log);
            default:
                throw std::invalid_argument("Unknown transaction type");
        }
    }
};

// ---------- ATM States ----------
class ATMState {
public:
    virtual ~ATMState() = default;
    virtual void insertCard(ATM& atm, const std::string& cardNumber) {
        std::cout << "Cannot insert card in current state\n";
    }
    virtual void enterPin(ATM& atm, const std::string& pin) {
        std::cout << "Cannot enter PIN in current state\n";
    }
    virtual void selectTransaction(ATM& atm, TransactionType type, double amount) {
        std::cout << "Cannot select transaction in current state\n";
    }
    virtual std::string name() const = 0;
};

class IdleState;
class HasCardState;
class AuthenticatedState;

// ---------- ATM (Context) ----------
class ATM {
    std::unique_ptr<ATMState> state;
    std::string currentCard;
    std::string currentAccount;
    BankServerProxy bankServer;
    CashDispenser dispenser;
    AuditLogger logger;

public:
    ATM();

    void setState(std::unique_ptr<ATMState> newState) { state = std::move(newState); }
    BankServerProxy& bank() { return bankServer; }
    CashDispenser& cashDispenser() { return dispenser; }
    AuditLogger& auditLogger() { return logger; }
    void setCurrentCard(const std::string& card) { currentCard = card; }
    void setCurrentAccount(const std::string& acct) { currentAccount = acct; }
    const std::string& account() const { return currentAccount; }

    void insertCard(const std::string& cardNumber) { state->insertCard(*this, cardNumber); }
    void enterPin(const std::string& pin) { state->enterPin(*this, pin); }
    void selectTransaction(TransactionType type, double amount) { state->selectTransaction(*this, type, amount); }
    std::string currentStateName() const { return state->name(); }
};

class IdleState : public ATMState {
public:
    void insertCard(ATM& atm, const std::string& cardNumber) override;
    std::string name() const override { return "Idle"; }
};

class HasCardState : public ATMState {
public:
    void enterPin(ATM& atm, const std::string& pin) override;
    std::string name() const override { return "HasCard"; }
};

class AuthenticatedState : public ATMState {
public:
    void selectTransaction(ATM& atm, TransactionType type, double amount) override;
    std::string name() const override { return "Authenticated"; }
};

void IdleState::insertCard(ATM& atm, const std::string& cardNumber) {
    std::cout << "Card inserted: " << cardNumber << "\n";
    atm.setCurrentCard(cardNumber);
    atm.setState(std::make_unique<HasCardState>());
}

void HasCardState::enterPin(ATM& atm, const std::string& pin) {
    if (atm.bank().authenticate("card-1", pin)) { // simplified: hardcoded card lookup
        atm.setCurrentAccount(atm.bank().accountFor("card-1"));
        std::cout << "PIN accepted\n";
        atm.setState(std::make_unique<AuthenticatedState>());
    } else {
        std::cout << "PIN incorrect\n";
        atm.setState(std::make_unique<IdleState>()); // simplified: no retry-count tracking shown
    }
}

void AuthenticatedState::selectTransaction(ATM& atm, TransactionType type, double amount) {
    auto txn = TransactionFactory::create(type, atm.account(), amount,
                                           atm.bank(), atm.cashDispenser(), atm.auditLogger());
    bool success = txn->execute();
    std::cout << (success ? "Transaction successful\n" : "Transaction failed\n");
    atm.setState(std::make_unique<IdleState>()); // back to idle, card ejected
}

ATM::ATM() { state = std::make_unique<IdleState>(); }

// ---------- Demo ----------
int main() {
    ATM atm;
    atm.insertCard("card-1");
    atm.enterPin("1234");
    atm.selectTransaction(TransactionType::BALANCE_INQUIRY, 0);

    atm.insertCard("card-1");
    atm.enterPin("1234");
    atm.selectTransaction(TransactionType::WITHDRAW, 700); // exceeds cash-friendly amount check example

    atm.insertCard("card-1");
    atm.enterPin("1234");
    atm.selectTransaction(TransactionType::WITHDRAW, 300);

    return 0;
}
```

---

## 10. Trade-offs

- **Local BankServerProxy vs true network calls** — a real ATM would need to handle network timeouts/retries to the bank server (see [[retry-and-timeout]]), with clear idempotency guarantees so a retried debit doesn't double-charge (see [[idempotency]]) — this demo simplifies that away.
- **State pattern adds classes but removes conditionals** — as the ATM gains more states (e.g. "card retained", "out of service"), the State pattern keeps each state's logic isolated at the cost of more classes to navigate; a simpler ATM with only 2-3 states might not need it (see [[state]] trade-offs).
- **Compensating credit on dispenser failure vs true distributed transaction** — the code above manually issues a compensating `credit()` if `dispense()` fails after `debit()` succeeds; a more robust design would use a proper Saga-style pattern (see [[acid]] section 6) with a durable log of in-flight transactions, so a crash between debit and dispense can be detected and reconciled on restart, not just handled if the crash doesn't happen.
- **Greedy denomination dispensing** — the demo's dispenser uses a simple greedy largest-denomination-first algorithm; this can fail to find an exact combination in some edge cases (e.g. needing $50 with only $100 notes available) even when the total cash on hand is sufficient — a real system needs a more careful (e.g. dynamic programming) denomination solver, or clearly-defined minimum withdrawal units.
- **Single ATM instance vs concurrent access to the same account** — this design doesn't address two ATMs (or an ATM and mobile banking) hitting the same account concurrently; the real BankServer must enforce atomic debit operations (e.g. `UPDATE ... WHERE balance >= amount` or row-level locking) — see [[transactions]], [[isolation-levels]].

---

## 11. Interview Questions

- **How do you ensure an account is never debited without cash being dispensed, or vice versa?** Treat debit + dispense as a single logical transaction: attempt debit first (cheap, reversible), then attempt dispense; if dispense fails, issue a compensating credit before reporting failure. For full robustness against crashes mid-sequence, persist an intent log (see [[acid]] Saga pattern) so a recovery process can detect and reconcile incomplete transactions after a restart.
- **How would you handle two ATMs (or an ATM and a mobile app) trying to withdraw from the same account simultaneously, and the balance is only enough for one?** This must be enforced at the BankServer/database layer, not the ATM — e.g. an atomic conditional update (`UPDATE accounts SET balance = balance - ? WHERE id = ? AND balance >= ?`), or row-level locking within a transaction (see [[transactions]], [[isolation-levels]]), so only one of the two concurrent debits succeeds.
- **Why use the State pattern here instead of a simple enum + switch statements?** As the number of states and operations grows, a single class with `switch(state)` blocks scattered across every method becomes hard to maintain and easy to leave inconsistent (forgetting to check state validity in one method). State pattern localizes each state's valid operations and transitions into its own class, making it structurally impossible to, say, accidentally allow "selectTransaction" while in `IdleState` — see [[state]].
- **How would you extend this design to support multiple accounts per card (e.g. checking and savings)?** Add an account-selection step between authentication and transaction selection — e.g. `AuthenticatedState` transitions to an `AccountSelectedState` after the customer picks which account to operate on, and `Transaction` objects reference that selected account rather than assuming one account per card.
- **What happens if the machine loses power mid-dispense?** This is the hardest real-world case — the debit may have already been committed to the bank server, but the physical cash dispensing was interrupted. Production ATMs handle this with hardware sensors confirming notes actually left the dispenser, and a reconciliation process comparing the bank's transaction log against the ATM's physical cash-count log to detect and resolve discrepancies (often requiring manual review).
- **How would you rate-limit or lock a card after repeated failed PIN attempts?** Track a failed-attempt counter per card (ideally on the BankServer side, since it must be consistent across ATMs, not just locally on one machine); after N consecutive failures, transition to a "card locked" state and require the customer to contact the bank rather than allowing further PIN attempts — a good place to reference [[rate-limiting]] concepts, applied to authentication attempts rather than API calls.

---

## Related Notes

- [[state]]
- [[factory-method]]
- [[proxy]]
- [[strategy]]
- [[singleton]]
- [[command]]
- [[acid]]
- [[transactions]]
- [[isolation-levels]]
- [[idempotency]]
- [[retry-and-timeout]]
- [[rate-limiting]]