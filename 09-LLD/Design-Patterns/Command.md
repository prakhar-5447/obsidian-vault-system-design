# Command Pattern

## Intent

Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations. Command decouples the object that **invokes** an operation from the object that knows **how to perform** it.

---

## Problem

You want to issue requests (e.g. "save file", "turn on light", "place order") without the invoker needing to know:

- **What** action will actually run, or **how** it's implemented
- The action's specific parameters/receiver ahead of time

Plain method calls hardcode the invoker directly to a concrete receiver and method — which breaks down when you need to:

- **Queue** requests to run later, or on a different thread
- **Log/persist** a history of requests (for replay or audit)
- Support **undo/redo** — a direct method call has no built-in way to reverse itself
- Let the **same UI trigger** (e.g. a generic button) run **different, swappable actions** without the button knowing what it does

---

## Solution

Wrap each request as a standalone **Command** object implementing a common interface (typically `execute()`, optionally `undo()`). The Command object stores everything needed to perform the action: a reference to the **Receiver** (the object that does the real work) and any parameters.

The **Invoker** holds and triggers Commands without knowing their concrete type or what they actually do — it just calls `execute()`. This is what decouples "what triggers an action" from "what the action does."

- **Simple Command** — directly calls one method on one receiver
- **Undoable Command** — also implements `undo()`, and the invoker keeps a history stack to support undo/redo
- **Macro Command** — a Command composed of a list of other Commands, executed in sequence (composes naturally with [[composite]])

---

## Structure

```text
┌────────────┐        ┌──────────────────┐        ┌──────────────────┐
│  Invoker   │───────▶│  Command (interface)│      │     Receiver     │
│ (holds and │        │  + execute()        │      │  (knows HOW to   │
│  triggers  │        │  + undo()            │      │   perform the    │
│  commands) │        └─────────┬─────────┘        │   actual action) │
└────────────┘                  │ implemented by     └────────▲─────────┘
                                 ▼                              │ calls
                       ┌──────────────────┐                    │
                       │  ConcreteCommand  │────────────────────┘
                       │  - receiver        │  (holds a reference to Receiver
                       │  + execute() {      │   and invokes the right method
                       │      receiver.action()│  with the right params)
                       │  }                   │
                       └──────────────────┘

Client creates ConcreteCommand(s), wires them to a Receiver, hands them to the Invoker.
Invoker only knows the Command interface — never the Receiver or concrete Command type.
```

---

## Example

```text
// Receiver — knows HOW to actually perform the action
class Light {
    private boolean isOn = false;

    void turnOn() {
        isOn = true;
        System.out.println("Light is ON");
    }

    void turnOff() {
        isOn = false;
        System.out.println("Light is OFF");
    }
}

// Command interface
interface Command {
    void execute();
    void undo();
}

// Concrete Command — binds a Receiver + action together
class TurnOnLightCommand implements Command {
    private final Light light;

    TurnOnLightCommand(Light light) {
        this.light = light;
    }

    @Override
    public void execute() {
        light.turnOn();
    }

    @Override
    public void undo() {
        light.turnOff(); // reverse of execute()
    }
}

class TurnOffLightCommand implements Command {
    private final Light light;

    TurnOffLightCommand(Light light) {
        this.light = light;
    }

    @Override
    public void execute() {
        light.turnOff();
    }

    @Override
    public void undo() {
        light.turnOn();
    }
}

// Invoker — knows nothing about Light or what a command actually does
class RemoteControl {
    private final Deque<Command> history = new ArrayDeque<>();

    void pressButton(Command command) {
        command.execute();
        history.push(command); // track for undo
    }

    void pressUndo() {
        if (!history.isEmpty()) {
            history.pop().undo();
        }
    }
}

// Client — wires Receiver, Commands, and Invoker together
Light livingRoomLight = new Light();
RemoteControl remote = new RemoteControl();

remote.pressButton(new TurnOnLightCommand(livingRoomLight));  // Light is ON
remote.pressButton(new TurnOffLightCommand(livingRoomLight)); // Light is OFF
remote.pressUndo();                                            // Light is ON (undo of turn-off)
```

---

## When to Use

- You need **undo/redo** functionality — Command is the standard way to make an operation reversible, since the command object can encapsulate both the forward action and its inverse
- You want to **queue, schedule, or log** requests for later/async execution (e.g. a job queue where each job is a Command object)
- You want to **parameterize** an object (like a UI button, menu item, or invoker) with an action to perform, without that object knowing the action's implementation
- You need to support **macro/composite operations** — batching several commands to run as one (e.g. "select all" + "delete" as a single undoable macro)
- You want a clean way to implement **transactional behavior** — commit is `execute()`, rollback is `undo()`

---

## When NOT to Use

- The action is simple, has a single fixed implementation, and never needs undo/queueing/logging — wrapping it in a Command class is unnecessary ceremony; just call the method directly
- Every additional action type means a **new Command class** — for a system with many one-off, rarely-reused actions, this can lead to significant class proliferation with little payoff
- The overhead of encapsulating state per-command (for undo) is expensive relative to the benefit — e.g. undoing an operation that touched a huge dataset may need a smarter approach (snapshots, event sourcing) than a simple in-memory undo stack

---

## Trade-offs

|Pros|Cons|
|---|---|
|Decouples invoker from receiver — invoker never needs to know how an action is implemented|Can lead to a proliferation of small Command classes, one per action|
|Enables undo/redo cleanly by encapsulating both do and undo logic together|Undo can be non-trivial for actions with side effects that are hard to reverse (e.g. sending an email, external API calls)|
|Commands are first-class objects — can be queued, logged, serialized, scheduled, or composed into macros|Adds indirection for what might otherwise be a single method call|
|Open/Closed — new commands can be added without changing the Invoker|Command objects holding receiver state can complicate memory/lifecycle management if history grows unbounded|

---

## Interview Questions

- **How is Command different from Strategy?** Both encapsulate a piece of behavior as an object, but the _intent_ differs: Strategy is about choosing **which algorithm/behavior** to use for a task (interchangeable implementations of the same operation). Command is about representing **a request itself** as an object — decoupling invoker from receiver, and typically supporting queuing/undo, which Strategy doesn't address at all.
- **How would you implement undo/redo with this pattern?** Each Command implements both `execute()` and `undo()` (the exact inverse operation). The Invoker maintains a history stack of executed commands; undo pops and calls `undo()` on the most recent one, redo would use a second stack for undone commands that can be re-executed.
- **How do you handle a Macro/Composite command?** A `MacroCommand` implements the same `Command` interface but internally holds a `List<Command>`; its `execute()` iterates and executes each sub-command in order, and its `undo()` typically undoes them in **reverse** order — this composes naturally with the [[composite]] pattern.
- **What if an action can't be cleanly undone (e.g. sending an email, charging a card)?** Not all commands are truly undoable — for side-effecting external actions, "undo" often means issuing a **compensating action** (e.g. a refund command) rather than a literal reversal; some systems simply mark such commands as non-undoable and disable undo for that history entry.
- **Where have you seen Command used in real systems/frameworks?** GUI frameworks' button/menu-action binding (`java.awt.event.ActionListener`, `javax.swing.Action`); job/task queue systems where each queued job is essentially a serialized Command; database transaction commit/rollback; Redux-style state management, where each dispatched "action" is conceptually a Command object describing a state change.
- **Is Command related to the Observer pattern?** Not directly, but they're often used together — e.g. a Command's `execute()` might notify [[observer|observers]] after performing its action, but the two patterns solve different problems (Command = encapsulate a request; Observer = notify dependents of a state change).

---

## Related Notes

- [[strategy]]
- [[composite]]
- [[observer]]
- [[chain-of-responsibility]]