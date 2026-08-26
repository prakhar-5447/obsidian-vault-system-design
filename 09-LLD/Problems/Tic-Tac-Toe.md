# Tic-Tac-Toe

## 1. Requirements

**Functional**

- Support an N x N board (default 3x3), configurable at game creation
- Two players take turns placing their symbol (X / O) on an empty cell
- After each move, check for a win (a full row, column, or diagonal of the same symbol) or a draw (board full, no winner)
- Reject invalid moves — occupied cell, out-of-bounds cell, move after game has ended, move out of turn
- Support querying the current board state and current game status (IN_PROGRESS / X_WON / O_WON / DRAW)
- Support starting a new game / resetting the board

**Non-Functional**

- Move validation and win-check should be efficient — avoid re-scanning the whole board on every move (important for large N)
- Extensible to more than 2 players or larger boards without a redesign
- Thread-safe if used in a multiplayer/server context (single game instance accessed by concurrent requests)
- Clear separation between game logic and any UI/IO layer (console, web, etc.)

---

## 2. Actors

- **Player** — a human or bot making moves, identified by a symbol
- **Game / Referee** — orchestrates turns, validates moves, and determines the outcome
- **Board** — holds and exposes the grid state

---

## 3. Use Cases

- Start a new game with 2 players
- Player makes a move at a given (row, col)
- System validates the move and updates the board
- System checks for a win/draw after every move
- System reports the game result when it ends
- System rejects an invalid move and lets the same player retry

---

## 4. Classes

- `Symbol` (enum) — `X`, `O`, `EMPTY`
- `GameStatus` (enum) — `IN_PROGRESS`, `X_WON`, `O_WON`, `DRAW`
- `Player` — holds a name and a `Symbol`
- `Board` — holds the `n x n` grid, exposes `placeSymbol()`, `isCellEmpty()`, `isFull()`
- `WinningStrategy` (interface) — checks whether the last move produced a win (open for different board sizes/win conditions)
- `Game` — orchestrates turn order, delegates move validation to `Board`, delegates win-check to `WinningStrategy`, tracks `GameStatus`
- `GameException` — custom exception for invalid moves (occupied cell, out of turn, game already over)

---

## 5. Relationships

- `Game` **has-a** `Board` (composition — board doesn't exist without the game)
- `Game` **has-a** list of `Player` (composition)
- `Game` **has-a** `WinningStrategy` (strategy pattern — pluggable win-checking logic)
- `Player` **has-a** `Symbol`
- `Board` **has-a** 2D grid of `Symbol`

---

## 6. Design Patterns

- **Strategy** — `WinningStrategy` decouples win-checking logic from `Game`, so different win conditions (standard row/col/diagonal, or a custom "4 in a row" rule for larger boards) can be swapped without changing `Game`
- **State** (optional extension) — `GameStatus` transitions (`IN_PROGRESS` → `X_WON`/`O_WON`/`DRAW`) can be modeled as a formal State pattern if the game grows more complex turn logic (e.g. undo, pause)
- **Singleton** (optional) — a `GameManager` coordinating multiple concurrent game instances could be a Singleton in a server context

---

## 7. Class Diagram

```text
┌────────────────┐        ┌────────────────────┐
│     Game        │───────▶│      Board          │
│ - board          │        │ - grid[n][n]          │
│ - players[]        │        │ - size                │
│ - currentPlayerIdx │        │ + placeSymbol(r,c,s) │
│ - status           │        │ + isCellEmpty(r,c)    │
│ - winStrategy       │        │ + isFull()             │
│ + makeMove(r,c)      │        └────────────────────┘
│ + getStatus()          │
└───────┬────────┘
        │ uses                          ┌────────────────────┐
        ▼                                │  <<interface>>      │
┌────────────────┐                     │  WinningStrategy      │
│     Player       │                     │ + checkWinner(board,  │
│ - name            │                     │     row, col, symbol) │
│ - symbol           │                     │     : boolean          │
└────────────────┘                     └──────────┬─────────┘
                                                    │ implemented by
                                       ┌─────────────────────────┐
                                       │  RowColDiagonalStrategy   │
                                       │ + checkWinner(...)          │
                                       └─────────────────────────┘
```

---

## 8. Sequence Flow

```text
Player X calls Game.makeMove(1, 1)
  │
  ▼
Game checks: status == IN_PROGRESS? currentPlayer == X?
  │  (fail → throw GameException, player retries)
  ▼
Game → Board.isCellEmpty(1,1)
  │  (false → throw GameException: cell occupied)
  ▼
Game → Board.placeSymbol(1, 1, X)
  │
  ▼
Game → WinningStrategy.checkWinner(board, 1, 1, X)
  │
  ├── true  → Game.status = X_WON → return result to caller
  ├── false, Board.isFull() → Game.status = DRAW → return result
  └── false, board not full → switch currentPlayer to O → await next move
```

---

## 9. Code

```cpp
#include <iostream>
#include <vector>
#include <stdexcept>
#include <memory>

enum class Symbol { EMPTY, X, O };
enum class GameStatus { IN_PROGRESS, X_WON, O_WON, DRAW };

class Board {
    int size;
    std::vector<std::vector<Symbol>> grid;
public:
    explicit Board(int n) : size(n), grid(n, std::vector<Symbol>(n, Symbol::EMPTY)) {}

    bool isCellEmpty(int r, int c) const {
        if (r < 0 || r >= size || c < 0 || c >= size)
            throw std::out_of_range("Cell out of bounds");
        return grid[r][c] == Symbol::EMPTY;
    }

    void placeSymbol(int r, int c, Symbol s) {
        if (!isCellEmpty(r, c)) throw std::logic_error("Cell already occupied");
        grid[r][c] = s;
    }

    bool isFull() const {
        for (auto& row : grid)
            for (auto& cell : row)
                if (cell == Symbol::EMPTY) return false;
        return true;
    }

    int getSize() const { return size; }
    Symbol get(int r, int c) const { return grid[r][c]; }
};

// Strategy pattern — pluggable win-checking logic
class WinningStrategy {
public:
    virtual bool checkWinner(const Board& board, int row, int col, Symbol s) = 0;
    virtual ~WinningStrategy() = default;
};

class RowColDiagonalStrategy : public WinningStrategy {
public:
    bool checkWinner(const Board& board, int row, int col, Symbol s) override {
        int n = board.getSize();
        bool rowWin = true, colWin = true, diagWin = true, antiDiagWin = true;

        for (int i = 0; i < n; ++i) {
            if (board.get(row, i) != s) rowWin = false;
            if (board.get(i, col) != s) colWin = false;
            if (board.get(i, i) != s) diagWin = false;
            if (board.get(i, n - 1 - i) != s) antiDiagWin = false;
        }
        return rowWin || colWin || diagWin || antiDiagWin;
    }
};

struct Player {
    std::string name;
    Symbol symbol;
};

class Game {
    Board board;
    std::vector<Player> players;
    int currentPlayerIdx = 0;
    GameStatus status = GameStatus::IN_PROGRESS;
    std::unique_ptr<WinningStrategy> winStrategy;

public:
    Game(int n, Player p1, Player p2, std::unique_ptr<WinningStrategy> strategy)
        : board(n), players{p1, p2}, winStrategy(std::move(strategy)) {}

    void makeMove(int row, int col) {
        if (status != GameStatus::IN_PROGRESS)
            throw std::logic_error("Game already ended");

        Player& current = players[currentPlayerIdx];
        board.placeSymbol(row, col, current.symbol);

        if (winStrategy->checkWinner(board, row, col, current.symbol)) {
            status = (current.symbol == Symbol::X) ? GameStatus::X_WON : GameStatus::O_WON;
        } else if (board.isFull()) {
            status = GameStatus::DRAW;
        } else {
            currentPlayerIdx = (currentPlayerIdx + 1) % players.size();
        }
    }

    GameStatus getStatus() const { return status; }
};

int main() {
    Player p1{"Alice", Symbol::X};
    Player p2{"Bob", Symbol::O};
    Game game(3, p1, p2, std::make_unique<RowColDiagonalStrategy>());

    game.makeMove(0, 0); // X
    game.makeMove(1, 0); // O
    game.makeMove(0, 1); // X
    game.makeMove(1, 1); // O
    game.makeMove(0, 2); // X wins top row

    if (game.getStatus() == GameStatus::X_WON) std::cout << "X wins!\n";
    return 0;
}
```

---

## 10. Trade-offs

- **Full board scan on every move (O(n) per check via strategy) vs incremental win tracking** — the row/col/diagonal counters approach (increment a counter per row/col/diagonal on each move, check if it hits `n`) is O(1) per move but adds bookkeeping complexity; the scan-based `WinningStrategy` shown above is simpler and fine for small boards, but doesn't scale well to very large N
- **Exceptions for invalid moves vs a return-code/`Result` type** — exceptions keep the happy path clean but are more expensive if invalid moves are frequent (e.g. in a bot-vs-bot simulation loop); a `MoveResult` enum return avoids exception overhead at the cost of the caller needing to check it explicitly
- **Strategy pattern for win-checking** adds a layer of indirection for what could be a single method, but pays off immediately if board size/win conditions need to vary (e.g. supporting a "4 in a row" variant on a larger board)
- **Mutable shared `Board` vs immutable board snapshots per move** — mutable is simpler and faster; immutable snapshots make undo/replay trivial but cost more memory per move

---

## 11. Interview Questions

- **How would you extend this to an N x N board with a "K in a row to win" rule (like Gomoku)?** Swap in a different `WinningStrategy` implementation that checks K-length runs through the last move's cell in all 4 directions, rather than requiring a full row/column/diagonal — the rest of the design (`Game`, `Board`, `Player`) doesn't need to change, which validates the Strategy pattern choice.
- **How would you make win-checking O(1) instead of O(n) per move?** Maintain running counters per row, column, and both diagonals — increment the relevant counters by +1/-1 depending on symbol after each move, and declare a win the instant a counter hits ±n. Trade-off: more state to maintain and keep in sync, vs recomputing from scratch each time.
- **How would you support more than 2 players?** Generalize `players` to a list of N, keep the same round-robin `currentPlayerIdx` rotation, and the `WinningStrategy` interface already works unchanged since it only checks against a given symbol — the interesting design question becomes what a "win" means with more players and more symbols.
- **How would you make this thread-safe for a multiplayer server handling concurrent move requests?** Guard `makeMove()` with a mutex/lock per game instance so moves are serialized; alternatively, use a single-threaded actor/queue model per game where all moves for that game are processed sequentially by one worker, avoiding lock contention entirely.
- **How would you support undo?** Have `Game` maintain a move history stack (could literally use the [[command]] pattern — each move is a Command with `execute()`/`undo()`); undo pops the last move, reverts the cell to `EMPTY`, decrements the counters (if using the O(1) approach), and rolls `currentPlayerIdx` back.