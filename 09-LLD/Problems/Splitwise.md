# Splitwise

## 1. Requirements

**Functional**

- Add users, each with a name, email, and a running balance ledger against other users
- Create an expense paid by one user and split among a group of users
- Support multiple split types: **Equal**, **Exact** (custom amounts per user), **Percentage**
- Track who owes whom, and how much — support querying a user's total balance and per-friend balance
- Support settling up (recording a payment between two users that reduces their mutual balance)
- Support group expenses (a group of users, e.g. a trip, with its own running balances)
- Simplify debts where possible (minimize the number of transactions needed to settle a group) — a common extension, not always required in v1

**Non-Functional**

- Balance calculations must be **consistent** — the sum of all balances across all users must always net to zero (money isn't created or destroyed)
- New split types should be addable without modifying existing expense logic (Open/Closed)
- Should handle **concurrent expense additions** correctly (two expenses added at the same time shouldn't corrupt balances)
- Reasonably efficient balance lookups — O(1) or close to it for "what do I owe this friend"

---

## 2. Actors

- **User** — a person who can pay for and be part of expenses
- **Group** — an optional collection of users sharing recurring expenses (e.g. roommates, a trip)
- **ExpenseManager** — orchestrates adding expenses and updating balances

---

## 3. Use Cases

- Register a new user
- Create a group and add users to it
- Add an expense: one payer, a total amount, a list of participants, and a split strategy
- System computes each participant's share based on the split type and updates pairwise balances
- User queries their balance with a specific friend, or their total net balance across everyone
- User settles up with a friend (records a payment, reduces the balance)
- User views their expense history

---

## 4. Classes

- `User` — id, name, email
- `Expense` — id, payer (`User`), amount, list of `Split`, description, timestamp
- `Split` (abstract) — a user's owed share of an expense; subclassed per split type
    - `EqualSplit`, `ExactSplit`, `PercentSplit`
- `SplitStrategy` (interface) — computes the list of `Split` given an amount and participants
    - `EqualSplitStrategy`, `ExactSplitStrategy`, `PercentSplitStrategy`
- `Group` — id, name, list of `User`, list of `Expense`
- `Balance` / `BalanceSheet` — tracks pairwise net amounts owed between users (e.g. `Map<User, Map<User, double>>`)
- `ExpenseManager` — facade coordinating `Expense` creation, `SplitStrategy` invocation, and `BalanceSheet` updates
- `Payment` — records a settle-up transaction between two users

---

## 5. Relationships

- `Expense` **has-a** payer (`User`) and a list of `Split` (composition)
- `Split` **is-a** abstract type, subclassed by `EqualSplit` / `ExactSplit` / `PercentSplit`
- `ExpenseManager` **uses-a** `SplitStrategy` (strategy pattern) to compute splits — chosen based on the expense's split type
- `ExpenseManager` **has-a** `BalanceSheet` (composition) — the single source of truth for who-owes-whom
- `Group` **has-many** `User` and **has-many** `Expense` (composition/aggregation)
- `ExpenseManager` acts as a **Facade** over the whole subsystem (`User`, `Group`, `Expense`, `Split`, `BalanceSheet`) for client code

---

## 6. Design Patterns

- **Strategy** — `SplitStrategy` decouples "how to compute each person's share" from `ExpenseManager`, so adding a new split type (e.g. "by shares/weights") means adding one new class, not touching existing logic
- **Facade** — `ExpenseManager` provides a simple `addExpense()` / `settleUp()` / `getBalance()` API over the more complex subsystem of splits, balances, and validation underneath
- **Observer** (optional extension) — notify users (push notification/email) whenever a new expense affects their balance, without `ExpenseManager` needing to know about the notification mechanism directly
- **Singleton** (optional) — `ExpenseManager` itself is often modeled as a Singleton in a single-process application, since there's conceptually one ledger authority

---

## 7. Class Diagram

```text
┌───────────────┐         ┌─────────────────────┐
│  ExpenseManager │────────▶│     BalanceSheet      │
│ + addExpense()    │         │ - balances[user][user] │
│ + settleUp()        │         │ + updateBalance(...)    │
│ + getBalance(u1,u2)   │         │ + getBalance(u1,u2)      │
└──────┬────────┘         └─────────────────────┘
       │ uses
       ▼
┌────────────────────┐        ┌────────────────┐
│  <<interface>>        │        │      User        │
│  SplitStrategy          │        │ - id, name, email │
│  + computeSplit(amount,   │        └────────────────┘
│      participants)         │
│      : List<Split>          │
└──────────┬─────────┘
           │ implemented by
 ┌──────────────┼──────────────┐
 ▼                ▼                ▼
┌──────────┐ ┌──────────┐ ┌────────────┐
│EqualSplit │ │ExactSplit │ │PercentSplit │
│Strategy    │ │Strategy    │ │Strategy      │
└──────────┘ └──────────┘ └────────────┘

┌────────────────┐        ┌────────────┐
│     Expense       │───────▶│    Split     │
│ - payer: User        │        │ - user        │
│ - amount               │        │ - amountOwed  │
│ - splits: List<Split>  │        └────────────┘
│ - description            │
└────────────────┘
```

---

## 8. Sequence Flow

```text
Client calls ExpenseManager.addExpense(payer=Alice, amount=300,
    participants=[Alice, Bob, Charlie], type=EQUAL)
  │
  ▼
ExpenseManager selects SplitStrategy → EqualSplitStrategy
  │
  ▼
EqualSplitStrategy.computeSplit(300, [Alice, Bob, Charlie])
  │  → [Split(Alice, 100), Split(Bob, 100), Split(Charlie, 100)]
  ▼
ExpenseManager creates Expense(payer=Alice, splits=[...])
  │
  ▼
For each Split where user != payer:
    BalanceSheet.updateBalance(payer=Alice, user=Bob, +100)   // Bob owes Alice 100
    BalanceSheet.updateBalance(payer=Alice, user=Charlie, +100) // Charlie owes Alice 100
  │
  ▼
ExpenseManager returns confirmation to client

--- Settle up ---
Client calls ExpenseManager.settleUp(from=Bob, to=Alice, amount=100)
  │
  ▼
BalanceSheet.updateBalance(Alice, Bob, -100) → balance now 0
```

---

## 9. Code

```cpp
#include <iostream>
#include <vector>
#include <unordered_map>
#include <memory>
#include <stdexcept>
#include <cmath>

struct User {
    int id;
    std::string name;
};

struct Split {
    User user;
    double amountOwed;
};

// Strategy pattern — pluggable split computation
class SplitStrategy {
public:
    virtual std::vector<Split> computeSplit(double amount, const std::vector<User>& participants,
                                              const std::vector<double>& values) = 0;
    virtual ~SplitStrategy() = default;
};

class EqualSplitStrategy : public SplitStrategy {
public:
    std::vector<Split> computeSplit(double amount, const std::vector<User>& participants,
                                     const std::vector<double>& /*unused*/) override {
        std::vector<Split> splits;
        double share = amount / participants.size();
        for (auto& u : participants) splits.push_back({u, share});
        return splits;
    }
};

class ExactSplitStrategy : public SplitStrategy {
public:
    std::vector<Split> computeSplit(double amount, const std::vector<User>& participants,
                                     const std::vector<double>& values) override {
        if (values.size() != participants.size())
            throw std::invalid_argument("Exact amounts must match participant count");
        double sum = 0;
        for (double v : values) sum += v;
        if (std::abs(sum - amount) > 0.01)
            throw std::invalid_argument("Exact amounts must sum to total");

        std::vector<Split> splits;
        for (size_t i = 0; i < participants.size(); ++i)
            splits.push_back({participants[i], values[i]});
        return splits;
    }
};

class PercentSplitStrategy : public SplitStrategy {
public:
    std::vector<Split> computeSplit(double amount, const std::vector<User>& participants,
                                     const std::vector<double>& percentages) override {
        double sum = 0;
        for (double p : percentages) sum += p;
        if (std::abs(sum - 100.0) > 0.01)
            throw std::invalid_argument("Percentages must sum to 100");

        std::vector<Split> splits;
        for (size_t i = 0; i < participants.size(); ++i)
            splits.push_back({participants[i], amount * percentages[i] / 100.0});
        return splits;
    }
};

// Balance sheet — pairwise net balances. Positive balances[a][b] means b owes a.
class BalanceSheet {
    std::unordered_map<int, std::unordered_map<int, double>> balances;
public:
    void updateBalance(const User& creditor, const User& debtor, double amount) {
        balances[creditor.id][debtor.id] += amount;
        balances[debtor.id][creditor.id] -= amount;
    }

    double getBalance(const User& a, const User& b) const {
        auto it = balances.find(a.id);
        if (it == balances.end()) return 0.0;
        auto it2 = it->second.find(b.id);
        return it2 == it->second.end() ? 0.0 : it2->second;
    }
};

// Facade
class ExpenseManager {
    BalanceSheet balanceSheet;
public:
    void addExpense(const User& payer, double amount, const std::vector<User>& participants,
                     std::unique_ptr<SplitStrategy> strategy, const std::vector<double>& values = {}) {
        std::vector<Split> splits = strategy->computeSplit(amount, participants, values);
        for (auto& split : splits) {
            if (split.user.id != payer.id) {
                balanceSheet.updateBalance(payer, split.user, split.amountOwed);
            }
        }
    }

    void settleUp(const User& from, const User& to, double amount) {
        balanceSheet.updateBalance(to, from, -amount); // reduces what 'from' owes 'to'
    }

    double getBalance(const User& a, const User& b) {
        return balanceSheet.getBalance(a, b);
    }
};

int main() {
    User alice{1, "Alice"}, bob{2, "Bob"}, charlie{3, "Charlie"};
    ExpenseManager manager;

    manager.addExpense(alice, 300, {alice, bob, charlie}, std::make_unique<EqualSplitStrategy>());

    std::cout << "Bob owes Alice: " << manager.getBalance(alice, bob) << "\n";     // 100
    std::cout << "Charlie owes Alice: " << manager.getBalance(alice, charlie) << "\n"; // 100

    manager.settleUp(bob, alice, 100);
    std::cout << "Bob owes Alice after settle: " << manager.getBalance(alice, bob) << "\n"; // 0

    return 0;
}
```

---

## 10. Trade-offs

- **Pairwise balance map (`user × user`) vs a ledger of raw transactions** — pairwise balances give O(1) balance lookups but lose per-expense history unless stored separately; a raw transaction ledger preserves full audit history but requires summing on every balance query (or maintaining both, at the cost of keeping them in sync)
- **Eager balance updates on `addExpense()` vs lazy computation on `getBalance()`** — eager (shown above) makes reads fast but writes slightly more work; lazy defers cost to reads, better if expenses are added far more often than balances are queried
- **Debt simplification (minimizing transaction count across a group)** is a genuinely separate, harder problem (greedy heap-based algorithm matching max creditor to max debtor) — worth explicitly scoping out of v1 and mentioning as a follow-up extension rather than conflating it with basic balance tracking
- **Floating-point currency arithmetic** — using `double` (as in the example) risks rounding errors accumulating over many splits; production systems typically store amounts as integer cents/minor units to avoid this entirely

---

## 11. Interview Questions

- **How would you implement "simplify debts" to minimize the number of settle-up transactions in a group?** Compute each user's net balance (total owed minus total owing) across the group, separate into creditors (positive) and debtors (negative), then greedily match the largest creditor with the largest debtor repeatedly (a max-heap on each side) — this minimizes the number of transactions needed to zero out the group, though it may reassign _who_ pays _whom_ differently than the original expenses.
- **How do you guarantee the sum of all balances always nets to zero?** By construction — every `updateBalance()` call adjusts both directions symmetrically (`+amount` for one user, `-amount` for the other), so the invariant is maintained at the operation level rather than needing a separate reconciliation step; still worth writing an invariant-check test that sums all balances after any operation.
- **How would you avoid floating-point rounding errors in an equal split that doesn't divide evenly (e.g. $100 among 3 people)?** Compute the integer base share and remainder in minor units (cents): `base = 10000 / 3 = 3333`, remainder `= 1`; distribute the extra cent(s) to the first N participants (or the payer) so the total still exactly sums to the original amount.
- **How would you extend this to support recurring/group expenses across a trip with many participants joining/leaving mid-trip?** Model `Group` as owning its own list of `Expense`s and a subset of `BalanceSheet` scoped to group members; a user joining later only affects balances from expenses added after they joined, which argues for timestamping both `User` group-membership and `Expense` creation.
- **How would you make expense creation safe under concurrent requests (two expenses added to the same pair of users at once)?** Lock at the granularity of the affected balance-sheet entries (or the whole `BalanceSheet` for simplicity) during an update, or use atomic increment operations backed by a database transaction if this were persisted — the in-memory map shown here is not thread-safe as written.
- **Why use the Strategy pattern for splits instead of an `if/else` on split type inside `addExpense()`?** Open/Closed — adding a new split type (e.g. split by shares/weights) means adding one new `SplitStrategy` implementation, with zero changes to `ExpenseManager` or existing strategies; an `if/else` chain would need modification every time a new type is added.