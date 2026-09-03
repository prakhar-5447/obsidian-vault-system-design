# Gang of Four (GoF) Design Patterns

The **Gang of Four (GoF)** design patterns are 23 classic
object-oriented design patterns described in *Design Patterns: Elements
of Reusable Object-Oriented Software* by Erich Gamma, Richard Helm,
Ralph Johnson, and John Vlissides.

They are grouped into three categories:

-   \[\[#Creational Patterns\|Creational\]\] --- how objects are created
-   \[\[#Structural Patterns\|Structural\]\] --- how objects/classes are
    composed
-   \[\[#Behavioral Patterns\|Behavioral\]\] --- how objects communicate
    and distribute responsibilities

> \[!tip\] Core idea A design pattern is not a library or framework. It
> is a reusable **design approach** for solving a recurring
> software-design problem.

------------------------------------------------------------------------

## Quick Reference

  -----------------------------------------------------------------------
  Category                Pattern                 Main Idea
  ----------------------- ----------------------- -----------------------
  Creational              \[\[#1. Singleton\]\]   One shared instance

  Creational              \[\[#2. Factory         Let subclasses decide
                          Method\]\]              what object to create

  Creational              \[\[#3. Abstract        Create related families
                          Factory\]\]             of objects

  Creational              \[\[#4. Builder\]\]     Construct complex
                                                  objects step by step

  Creational              \[\[#5. Prototype\]\]   Create objects by
                                                  cloning existing ones

  Structural              \[\[#6. Adapter\]\]     Make incompatible
                                                  interfaces work
                                                  together

  Structural              \[\[#7. Bridge\]\]      Separate abstraction
                                                  from implementation

  Structural              \[\[#8. Composite\]\]   Treat individual and
                                                  grouped objects
                                                  uniformly

  Structural              \[\[#9. Decorator\]\]   Add behavior without
                                                  changing the original
                                                  class

  Structural              \[\[#10. Facade\]\]     Provide a simple
                                                  interface to a complex
                                                  subsystem

  Structural              \[\[#11. Flyweight\]\]  Share common state to
                                                  reduce memory usage

  Structural              \[\[#12. Proxy\]\]      Control access to
                                                  another object

  Behavioral              \[\[#13. Chain of       Pass a request through
                          Responsibility\]\]      possible handlers

  Behavioral              \[\[#14. Command\]\]    Encapsulate a request
                                                  as an object

  Behavioral              \[\[#15.                Represent and evaluate
                          Interpreter\]\]         a grammar

  Behavioral              \[\[#16. Iterator\]\]   Traverse a collection
                                                  without exposing
                                                  internals

  Behavioral              \[\[#17. Mediator\]\]   Centralize
                                                  communication between
                                                  objects

  Behavioral              \[\[#18. Memento\]\]    Capture and restore
                                                  object state

  Behavioral              \[\[#19. Observer\]\]   Notify dependents when
                                                  state changes

  Behavioral              \[\[#20. State\]\]      Change behavior when
                                                  internal state changes

  Behavioral              \[\[#21. Strategy\]\]   Make an algorithm
                                                  interchangeable

  Behavioral              \[\[#22. Template       Define an algorithm
                          Method\]\]              skeleton with
                                                  customizable steps

  Behavioral              \[\[#23. Visitor\]\]    Add operations to
                                                  object structures
                                                  without changing their
                                                  classes
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# Creational Patterns

Creational patterns focus on **object creation**.

They help avoid tightly coupling application code to concrete classes.

------------------------------------------------------------------------

## 1. Singleton

### Intent

Ensure a class has **only one instance** and provide a global access
point to it.

### Problem

Multiple parts of an application need access to the same resource, and
creating multiple instances would be incorrect or wasteful.

### Structure

``` text
Client
  |
  v
Singleton
  |
  +-- private constructor
  +-- static instance
  +-- getInstance()
```

### Example

``` java
public final class ConfigManager {
    private static ConfigManager instance;

    private ConfigManager() {}

    public static synchronized ConfigManager getInstance() {
        if (instance == null) {
            instance = new ConfigManager();
        }
        return instance;
    }
}
```

### Use Cases

-   Configuration manager
-   Application-wide cache
-   Logger
-   Shared resource manager

### Trade-offs

**Pros** - Guarantees a single instance - Centralized access

**Cons** - Introduces global state - Can make testing harder - Often
overused

> \[!warning\] Design note Dependency injection is often preferable to
> Singleton because dependencies remain explicit and easier to test.

------------------------------------------------------------------------

## 2. Factory Method

### Intent

Define an interface for creating an object, but let subclasses decide
which concrete object to instantiate.

### Problem

A class needs to create objects, but it should not depend directly on
every concrete implementation.

### Example

``` java
interface Notification {
    void send(String message);
}

class EmailNotification implements Notification {
    public void send(String message) {
        System.out.println("Email: " + message);
    }
}

class SmsNotification implements Notification {
    public void send(String message) {
        System.out.println("SMS: " + message);
    }
}

abstract class NotificationService {
    protected abstract Notification createNotification();

    public void notifyUser(String message) {
        Notification notification = createNotification();
        notification.send(message);
    }
}

class EmailService extends NotificationService {
    protected Notification createNotification() {
        return new EmailNotification();
    }
}
```

### Key Idea

``` text
NotificationService
        |
        | createNotification()
        v
   Notification
      /     \
   Email    SMS
```

### Use When

-   Object creation varies by subclass
-   You want to depend on abstractions
-   New product types may be added

------------------------------------------------------------------------

## 3. Abstract Factory

### Intent

Provide an interface for creating **families of related objects**
without specifying their concrete classes.

### Factory Method vs Abstract Factory

``` text
Factory Method
    -> usually creates one product

Abstract Factory
    -> creates a family of related products
```

### Example

``` java
interface Button {
    void render();
}

interface Checkbox {
    void render();
}

interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

class WindowsFactory implements UIFactory {
    public Button createButton() {
        return new WindowsButton();
    }

    public Checkbox createCheckbox() {
        return new WindowsCheckbox();
    }
}
```

### Use Cases

-   Cross-platform UI
-   Database families
-   Theme systems
-   Cloud-provider-specific components

------------------------------------------------------------------------

## 4. Builder

### Intent

Construct a complex object **step by step**, allowing different
representations of the same construction process.

### Problem

A constructor becomes difficult to use when an object has many optional
parameters.

### Example

``` java
class User {
    private final String name;
    private final String email;
    private final int age;

    private User(Builder builder) {
        this.name = builder.name;
        this.email = builder.email;
        this.age = builder.age;
    }

    public static class Builder {
        private String name;
        private String email;
        private int age;

        public Builder name(String name) {
            this.name = name;
            return this;
        }

        public Builder email(String email) {
            this.email = email;
            return this;
        }

        public Builder age(int age) {
            this.age = age;
            return this;
        }

        public User build() {
            return new User(this);
        }
    }
}
```

Usage:

``` java
User user = new User.Builder()
        .name("Prakhar")
        .email("user@example.com")
        .age(24)
        .build();
```

### Use When

-   Many optional fields
-   Immutable objects
-   Construction has multiple steps
-   Readable object creation matters

------------------------------------------------------------------------

## 5. Prototype

### Intent

Create new objects by **copying an existing object** rather than
constructing from scratch.

### Example

``` java
interface Shape extends Cloneable {
    Shape clone();
}

class Circle implements Shape {
    private int radius;

    public Circle(int radius) {
        this.radius = radius;
    }

    public Shape clone() {
        return new Circle(radius);
    }
}
```

### Use When

-   Object creation is expensive
-   Initial configuration is complex
-   Many objects share a similar starting state

### Important

Be careful with **shallow copy vs deep copy** when objects contain
references to mutable objects.

------------------------------------------------------------------------

# Structural Patterns

Structural patterns focus on **how classes and objects are composed**.

------------------------------------------------------------------------

## 6. Adapter

### Intent

Convert the interface of one class into another interface that clients
expect.

### Real-world analogy

A power adapter lets a device with one plug type work with a different
socket.

### Example

``` java
interface PaymentProcessor {
    void pay(double amount);
}

class LegacyPaymentGateway {
    void makePayment(double amount) {
        System.out.println("Paid " + amount);
    }
}

class PaymentAdapter implements PaymentProcessor {
    private final LegacyPaymentGateway gateway;

    PaymentAdapter(LegacyPaymentGateway gateway) {
        this.gateway = gateway;
    }

    public void pay(double amount) {
        gateway.makePayment(amount);
    }
}
```

### Key Idea

``` text
Client
  |
  v
Expected Interface
  |
  v
Adapter
  |
  v
Existing Incompatible Class
```

### Use When

-   Integrating legacy code
-   Integrating third-party APIs
-   Migrating interfaces incrementally

------------------------------------------------------------------------

## 7. Bridge

### Intent

Decouple an abstraction from its implementation so that both can vary
independently.

### Example

``` text
RemoteControl          Device
     |                   |
     v                   v
   Abstraction ------> Implementation
```

For example:

``` text
RemoteControl
    |
    +---- TV
    |
    +---- Radio

Device
    |
    +---- Sony
    +---- Samsung
```

Without Bridge, combinations can lead to many subclasses.

### Use When

-   Two dimensions of variation exist
-   Inheritance would create a class explosion
-   Abstraction and implementation should evolve independently

------------------------------------------------------------------------

## 8. Composite

### Intent

Compose objects into tree structures and treat **individual objects and
compositions uniformly**.

### Example

A file system:

``` text
Directory
├── File
├── File
└── Directory
    ├── File
    └── File
```

### Code Shape

``` java
interface Component {
    void show();
}

class File implements Component {
    public void show() {
        System.out.println("File");
    }
}

class Directory implements Component {
    private List<Component> children = new ArrayList<>();

    public void add(Component component) {
        children.add(component);
    }

    public void show() {
        for (Component child : children) {
            child.show();
        }
    }
}
```

### Use Cases

-   File systems
-   UI component trees
-   Organization hierarchies
-   Menu structures
-   ASTs

------------------------------------------------------------------------

## 9. Decorator

### Intent

Attach additional responsibilities to an object dynamically without
modifying its class.

### Example

``` text
Coffee
  |
  v
MilkDecorator
  |
  v
SugarDecorator
```

``` java
interface Coffee {
    double cost();
}

class BasicCoffee implements Coffee {
    public double cost() {
        return 100;
    }
}

class MilkDecorator implements Coffee {
    private final Coffee coffee;

    MilkDecorator(Coffee coffee) {
        this.coffee = coffee;
    }

    public double cost() {
        return coffee.cost() + 20;
    }
}
```

### Decorator vs Inheritance

Inheritance:

``` text
Coffee
 ├── MilkCoffee
 ├── SugarCoffee
 └── MilkSugarCoffee
```

Decorator:

``` text
Coffee
  -> Milk
  -> Sugar
```

Decorator avoids a large number of subclasses.

------------------------------------------------------------------------

## 10. Facade

### Intent

Provide a **simple interface** to a complex subsystem.

### Example

``` text
Client
  |
  v
OrderFacade
  |
  +--> InventoryService
  +--> PaymentService
  +--> NotificationService
  +--> ShippingService
```

The client only calls:

``` java
orderFacade.placeOrder(order);
```

instead of coordinating every subsystem itself.

### Use When

-   A subsystem is complex
-   Clients need a simplified API
-   You want to reduce coupling

> \[!tip\] Facade does not necessarily hide the subsystem completely.
> The underlying services can still be accessible when advanced clients
> need them.

------------------------------------------------------------------------

## 11. Flyweight

### Intent

Use sharing to efficiently support large numbers of fine-grained
objects.

### Core Idea

Separate:

-   **Intrinsic state** --- shared
-   **Extrinsic state** --- supplied by the client

### Example

A text editor can share font objects:

``` text
Character A -> Font("Arial", 12)
Character B -> Font("Arial", 12)
Character C -> Font("Arial", 12)
```

Instead of storing a separate font object for every character.

### Use When

-   Huge numbers of similar objects exist
-   Memory usage is a concern
-   Much of the object state can be shared

------------------------------------------------------------------------

## 12. Proxy

### Intent

Provide a substitute or placeholder that controls access to another
object.

### Common Types

-   Virtual proxy --- lazy loading
-   Protection proxy --- authorization
-   Remote proxy --- remote service
-   Caching proxy --- cache results
-   Logging proxy --- add logging

### Example

``` text
Client
  |
  v
Proxy
  |
  +--> check access
  +--> cache
  +--> log
  |
  v
Real Service
```

### Proxy vs Decorator

**Proxy** - Controls access to an object

**Decorator** - Adds responsibilities/behavior

They may look structurally similar, but their intent differs.

------------------------------------------------------------------------

# Behavioral Patterns

Behavioral patterns focus on **communication, algorithms, and
responsibility distribution**.

------------------------------------------------------------------------

## 13. Chain of Responsibility

### Intent

Pass a request along a chain of handlers until one handles it.

### Example

``` text
Request
   |
   v
AuthHandler
   |
   v
ValidationHandler
   |
   v
PaymentHandler
   |
   v
OrderHandler
```

Each handler decides:

``` text
Can I handle it?
   |
   +-- Yes -> handle
   |
   +-- No --> next handler
```

### Use Cases

-   HTTP middleware
-   Authentication/authorization pipelines
-   Validation
-   Logging
-   Approval workflows

------------------------------------------------------------------------

## 14. Command

### Intent

Encapsulate a request as an object.

### Structure

``` text
Invoker -> Command -> Receiver
```

### Example

``` java
interface Command {
    void execute();
}

class Light {
    void on() {
        System.out.println("Light ON");
    }
}

class LightOnCommand implements Command {
    private final Light light;

    LightOnCommand(Light light) {
        this.light = light;
    }

    public void execute() {
        light.on();
    }
}
```

### Benefits

Commands can be:

-   queued
-   logged
-   retried
-   undone
-   scheduled

### Use Cases

-   Undo/redo
-   Job queues
-   GUI actions
-   Transactions
-   Task scheduling

------------------------------------------------------------------------

## 15. Interpreter

### Intent

Define a representation for a language's grammar and provide an
interpreter for sentences in that language.

### Example

``` text
Expression
├── NumberExpression
├── AddExpression
├── SubtractExpression
└── MultiplyExpression
```

For:

``` text
10 + 5
```

the expression tree could be:

``` text
    +
   / \
  10  5
```

### Use When

-   Grammar is simple
-   Expressions need to be evaluated
-   Rules are relatively stable

### Examples

-   Simple query languages
-   Rule engines
-   Mathematical expressions

> \[!note\] For complex grammars, parser generators or dedicated parsing
> libraries are generally more appropriate.

------------------------------------------------------------------------

## 16. Iterator

### Intent

Provide a way to access elements of a collection sequentially without
exposing its underlying representation.

### Example

``` java
Iterator<String> iterator = users.iterator();

while (iterator.hasNext()) {
    System.out.println(iterator.next());
}
```

### Key Idea

The client knows:

``` text
hasNext()
next()
```

but does not need to know whether the collection uses:

-   Array
-   Linked list
-   Tree
-   Hash-based structure

------------------------------------------------------------------------

## 17. Mediator

### Intent

Define an object that encapsulates how a set of objects interact.

### Without Mediator

``` text
A <--> B
A <--> C
A <--> D
B <--> C
B <--> D
C <--> D
```

Communication becomes tightly coupled.

### With Mediator

``` text
      A
      |
B --> Mediator <-- C
      |
      D
```

Objects communicate through the mediator.

### Use Cases

-   Chat rooms
-   UI component coordination
-   Workflow engines
-   Air traffic control systems

------------------------------------------------------------------------

## 18. Memento

### Intent

Capture and externalize an object's internal state so it can later be
restored without violating encapsulation.

### Example

``` text
Editor
  |
  | save()
  v
Memento
  |
  | restore()
  v
Editor
```

### Common Use Case

Undo functionality:

``` text
State 1
  |
save
  v
Memento 1

State 2
  |
save
  v
Memento 2

Undo -> restore Memento 1
```

### Participants

-   **Originator** --- object whose state is saved
-   **Memento** --- stores state
-   **Caretaker** --- manages mementos

------------------------------------------------------------------------

## 19. Observer

### Intent

Define a one-to-many dependency so that when one object changes state,
all dependent objects are notified.

### Structure

``` text
        Subject
       /   |   \
      v    v    v
 Observer Observer Observer
```

### Example

``` java
interface Observer {
    void update(String message);
}

class NotificationObserver implements Observer {
    public void update(String message) {
        System.out.println(message);
    }
}
```

### Use Cases

-   Event systems
-   UI state updates
-   Pub/sub systems
-   Notifications

### Observer vs Pub/Sub

**Observer** - Usually direct object-to-object relationship

**Pub/Sub** - Usually uses an intermediary broker/event bus

------------------------------------------------------------------------

## 20. State

### Intent

Allow an object to alter its behavior when its internal state changes.

### Example

An order:

``` text
Order
 |
 +--> Created
 |
 +--> Paid
 |
 +--> Shipped
 |
 +--> Delivered
 |
 +--> Cancelled
```

Instead of:

``` java
if (status == CREATED) ...
else if (status == PAID) ...
else if (status == SHIPPED) ...
```

behavior can be moved into state objects.

### Structure

``` text
Context
  |
  v
State
 / \
Created Paid
```

### Use When

-   Object behavior changes significantly based on state
-   Large conditional statements are growing
-   State transitions need clear boundaries

------------------------------------------------------------------------

## 21. Strategy

### Intent

Define a family of algorithms, encapsulate each one, and make them
interchangeable.

### Example

Payment strategies:

``` text
PaymentService
      |
      v
 PaymentStrategy
   /    |     \
UPI   Card   Wallet
```

``` java
interface PaymentStrategy {
    void pay(double amount);
}

class UpiPayment implements PaymentStrategy {
    public void pay(double amount) {
        System.out.println("Pay using UPI");
    }
}

class CardPayment implements PaymentStrategy {
    public void pay(double amount) {
        System.out.println("Pay using Card");
    }
}
```

### Strategy vs State

They often have the same structure but different intent.

**Strategy** - Client chooses the algorithm

**State** - Object's state determines behavior

------------------------------------------------------------------------

## 22. Template Method

### Intent

Define the skeleton of an algorithm in a base class while allowing
subclasses to redefine certain steps.

### Example

``` text
processOrder()
    |
    +--> validate()
    +--> prepare()
    +--> executePayment()
    +--> notify()
```

The overall sequence stays fixed while individual steps can vary.

### Structure

``` java
abstract class DataProcessor {

    public final void process() {
        readData();
        processData();
        saveData();
    }

    abstract void readData();
    abstract void processData();

    void saveData() {
        System.out.println("Saving data");
    }
}
```

### Use When

-   Algorithm structure is fixed
-   Some steps vary
-   You want to avoid duplicated workflow code

------------------------------------------------------------------------

## 23. Visitor

### Intent

Represent an operation to be performed on elements of an object
structure, allowing new operations without changing the element classes.

### Structure

``` text
Visitor
  |
  +--> visit(File)
  +--> visit(Directory)

Element
  |
  +--> File
  +--> Directory
```

### Example Use Case

A file system might support:

-   Calculate size
-   Export metadata
-   Virus scan
-   Generate report

The file/directory classes remain unchanged while visitors implement
operations.

### Use When

-   Object structure is relatively stable
-   Many unrelated operations need to be performed
-   Adding operations is more common than adding element types

### Trade-off

Visitor makes adding **new operations** easy but adding **new element
types** harder.

------------------------------------------------------------------------

# Pattern Relationships

Understanding similar patterns is often more useful than memorizing
definitions.

## Factory Method vs Abstract Factory

  -----------------------------------------------------------------------
                          Factory Method          Abstract Factory
  ----------------------- ----------------------- -----------------------
  Main goal               Create a product        Create a product family

  Typical scope           One creation method     Multiple related
                                                  creation methods

  Example                 Create `Notification`   Create complete UI
                                                  theme

  Main mechanism          Often inheritance       Often composition
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## Adapter vs Facade

  -----------------------------------------------------------------------
                          Adapter                 Facade
  ----------------------- ----------------------- -----------------------
  Purpose                 Compatibility           Simplification

  Interfaces              Converts one interface  Provides a simpler
                          to another              interface

  Existing API            Usually incompatible    Usually complex

  Client view             Makes existing object   Makes subsystem easier
                          usable                  to use
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## Decorator vs Proxy

  -----------------------------------------------------------------------
                          Decorator               Proxy
  ----------------------- ----------------------- -----------------------
  Intent                  Add behavior            Control access

  Typical behavior        Logging, compression,   Auth, lazy loading,
                          features                remote access

  Object identity         Wraps same conceptual   Stands in for target
                          service                 
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## Strategy vs State

                 Strategy                   State
  -------------- -------------------------- -----------------------------
  Focus          Algorithm                  Object lifecycle/state
  Who chooses?   Client/application         Context/state transitions
  Goal           Interchangeable behavior   Behavior changes with state

------------------------------------------------------------------------

## Composite vs Decorator

Both can wrap components.

**Composite** - Represents a tree - Has children - Treats leaf and group
uniformly

**Decorator** - Wraps one component - Adds behavior - Usually preserves
the same interface

------------------------------------------------------------------------

## Observer vs Mediator

**Observer** - One subject notifies many observers

**Mediator** - Central object coordinates communication among many peers

------------------------------------------------------------------------

# How to Choose a GoF Pattern

Start with the problem rather than the pattern name.

``` text
Need to create objects?
        |
        +--> One instance? --------> Singleton
        |
        +--> Complex construction? -> Builder
        |
        +--> Clone existing object? -> Prototype
        |
        +--> Creation varies? ------> Factory Method
        |
        +--> Related object family? -> Abstract Factory
```

``` text
Need to compose objects?
        |
        +--> Incompatible interface? -> Adapter
        |
        +--> Add behavior dynamically? -> Decorator
        |
        +--> Simplify subsystem? -----> Facade
        |
        +--> Control access? ---------> Proxy
        |
        +--> Tree structure? ---------> Composite
        |
        +--> Separate abstraction/
             implementation? --------> Bridge
```

``` text
Need to manage behavior?
        |
        +--> Interchangeable algorithm? -> Strategy
        |
        +--> State-dependent behavior? -> State
        |
        +--> Request pipeline? --------> Chain of Responsibility
        |
        +--> Request as object? --------> Command
        |
        +--> Notify dependents? --------> Observer
        |
        +--> Coordinate many objects? -> Mediator
        |
        +--> Undo/restore state? ------> Memento
```

------------------------------------------------------------------------

# The 23 Patterns at a Glance

## Creational

1.  **Singleton** --- one instance
2.  **Factory Method** --- subclasses decide creation
3.  **Abstract Factory** --- families of related objects
4.  **Builder** --- step-by-step construction
5.  **Prototype** --- cloning

## Structural

6.  **Adapter** --- compatibility
7.  **Bridge** --- separate abstraction and implementation
8.  **Composite** --- tree structures
9.  **Decorator** --- dynamic behavior
10. **Facade** --- simplified interface
11. **Flyweight** --- shared objects
12. **Proxy** --- controlled access

## Behavioral

13. **Chain of Responsibility** --- request pipeline
14. **Command** --- request as object
15. **Interpreter** --- grammar/expression evaluation
16. **Iterator** --- sequential traversal
17. **Mediator** --- centralized communication
18. **Memento** --- save/restore state
19. **Observer** --- state-change notification
20. **State** --- behavior based on state
21. **Strategy** --- interchangeable algorithms
22. **Template Method** --- algorithm skeleton
23. **Visitor** --- operations over object structures

------------------------------------------------------------------------

# Practical Rule of Thumb

When designing a system, ask:

### Creation

> **"Who should create this object?"**

Think:

`Factory Method` · `Abstract Factory` · `Builder` · `Prototype`

### Composition

> **"How can these objects work together without tight coupling?"**

Think:

`Adapter` · `Bridge` · `Composite` · `Decorator` · `Facade` · `Proxy`

### Behavior

> **"Who should own this behavior, and how should objects
> communicate?"**

Think:

`Strategy` · `State` · `Observer` · `Command` · `Mediator` ·
`Chain of Responsibility`

------------------------------------------------------------------------

# Important Design Principle

Do not use a GoF pattern simply because it exists.

A good design usually follows:

``` text
Problem
   ↓
Constraints
   ↓
Simple design
   ↓
Identify repeated structure
   ↓
Apply pattern if it reduces complexity
```

Patterns are **tools**, not goals.

> \[!important\] Prefer the simplest design that satisfies the
> requirements. Introduce a pattern when it makes the code easier to
> extend, understand, test, or maintain.

------------------------------------------------------------------------

# Recommended Learning Order

For practical software development, a useful order is:

1.  \[\[#Strategy\]\]
2.  \[\[#Factory Method\]\]
3.  \[\[#Builder\]\]
4.  \[\[#Observer\]\]
5.  \[\[#Decorator\]\]
6.  \[\[#Adapter\]\]
7.  \[\[#Facade\]\]
8.  \[\[#State\]\]
9.  \[\[#Command\]\]
10. \[\[#Chain of Responsibility\]\]
11. \[\[#Composite\]\]
12. \[\[#Proxy\]\]
13. \[\[#Abstract Factory\]\]
14. \[\[#Template Method\]\]
15. \[\[#Mediator\]\]
16. \[\[#Iterator\]\]
17. \[\[#Singleton\]\]
18. \[\[#Prototype\]\]
19. \[\[#Bridge\]\]
20. \[\[#Flyweight\]\]
21. \[\[#Memento\]\]
22. \[\[#Visitor\]\]
23. \[\[#Interpreter\]\]

------------------------------------------------------------------------

# Obsidian Notes Structure

A good vault structure for deeper study:

``` text
Design Patterns/
├── GoF/
│   ├── GoF - Overview.md
│   ├── Creational/
│   │   ├── Singleton.md
│   │   ├── Factory Method.md
│   │   ├── Abstract Factory.md
│   │   ├── Builder.md
│   │   └── Prototype.md
│   ├── Structural/
│   │   ├── Adapter.md
│   │   ├── Bridge.md
│   │   ├── Composite.md
│   │   ├── Decorator.md
│   │   ├── Facade.md
│   │   ├── Flyweight.md
│   │   └── Proxy.md
│   └── Behavioral/
│       ├── Chain of Responsibility.md
│       ├── Command.md
│       ├── Interpreter.md
│       ├── Iterator.md
│       ├── Mediator.md
│       ├── Memento.md
│       ├── Observer.md
│       ├── State.md
│       ├── Strategy.md
│       ├── Template Method.md
│       └── Visitor.md
```

## Related Concepts

-   \[\[SOLID Principles\]\]
-   \[\[Object-Oriented Design\]\]
-   \[\[Low Level Design\]\]
-   \[\[Composition over Inheritance\]\]
-   \[\[Dependency Injection\]\]
-   \[\[Loose Coupling\]\]
-   \[\[High Cohesion\]\]
-   \[\[Encapsulation\]\]
-   \[\[Polymorphism\]\]

# Summary

The GoF patterns can be remembered through three questions:

> **Creational:** How should I create objects?

> **Structural:** How should I compose objects?

> **Behavioral:** How should objects communicate and vary their
> behavior?

The real skill is not memorizing all 23 patterns. It is recognizing the
**design problem** and selecting the simplest pattern that addresses it.
