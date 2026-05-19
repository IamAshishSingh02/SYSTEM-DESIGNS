# Head First Design Patterns — Fast-Read Study Guide

*Interview-ready notes for OOP and System Design rounds, based on Freeman & Robson (O'Reilly, 2nd ed., 2020).*

---

## How to use this guide

For each concept you get:
1. **TL;DR** — one or two lines, the "elevator pitch" you'd give an interviewer.
2. **The problem it solves** — the pain point that motivates the pattern.
3. **Structure** — the moving parts.
4. **Real-world / Java examples** — where you've already seen it.
5. **Interview angles** — common follow-ups, trade-offs, gotchas.

Patterns are grouped by the **GoF taxonomy** (Creational / Structural / Behavioral), not the book's order, because that's what interviewers expect.

---

# Part 1 — The Core OO Design Principles

Patterns are just the famous *solutions*. Interviewers love the *principles* because they show you can reason from scratch. Memorize these — they're how you'll defend any design decision.

### 1. Encapsulate what varies
Identify the parts of your app that change, separate them from what stays the same. Then you can modify the changing parts without touching the stable parts. *This is the meta-principle behind nearly every pattern.*

### 2. Favor composition over inheritance ("HAS-A over IS-A")
Inheritance is tightly coupled and decided at compile time. Composition (holding a reference to another object) is flexible and can be swapped at runtime. *Almost every "modern" pattern leans on composition.*

### 3. Program to an interface, not an implementation
Depend on abstractions (`List`), not concretes (`ArrayList`). Lets you swap implementations without touching client code. The Java idiom `List<String> l = new ArrayList<>();` is this principle in one line.

### 4. Strive for loosely coupled designs between objects that interact
Objects should know as little as possible about each other. This shows up explicitly in **Observer**.

### 5. Open-Closed Principle (OCP)
Classes should be **open for extension, closed for modification**. You add new behavior by writing new classes, not by editing old ones. Headlined in **Decorator**.

### 6. Dependency Inversion Principle (DIP)
Depend on abstractions, not on concrete classes. **Both** high-level modules and low-level modules depend on interfaces. Headlined in **Abstract Factory**.

### 7. Principle of Least Knowledge (Law of Demeter)
A method should only call methods on: itself, its parameters, objects it creates, and its direct components. Avoid chains like `a.getB().getC().doSomething()`. Headlined in **Facade**.

### 8. Hollywood Principle
*"Don't call us, we'll call you."* High-level components decide when low-level ones get invoked, not the other way around. Headlined in **Template Method** and **Observer**.

### 9. Single Responsibility Principle (SRP)
A class should have **one reason to change** — one job. Headlined in **Iterator** (separating "how to iterate" from "what's being iterated").

> **Interview tip:** When asked "why is this design good/bad?", reach for these principles by name. Saying *"this violates the Open-Closed Principle because adding a new payment type forces me to edit the existing PaymentProcessor"* is exactly the language interviewers want to hear.

---

# Part 2 — Creational Patterns
*How objects get created. Decouple the "what to create" from the "how to use it".*

## Singleton — Chapter 5
**TL;DR:** Ensure a class has exactly one instance and provide a global access point to it.

**Problem:** Some resources are inherently single — a config registry, a thread pool, a connection pool, a logger. You don't want two instances fighting over the same resource.

**Structure (classic Java):**
```java
public class Singleton {
    private static Singleton instance;
    private Singleton() {}                      // private constructor
    public static synchronized Singleton getInstance() {
        if (instance == null) instance = new Singleton();
        return instance;
    }
}
```

**Variants you must know:**
- **Eager initialization** — `private static final Singleton INSTANCE = new Singleton();`. Simple, thread-safe, but creates the object even if never used.
- **Lazy + synchronized method** — thread-safe but slow (every call grabs the lock).
- **Double-Checked Locking with `volatile`** — fast and lazy:
  ```java
  private static volatile Singleton instance;
  public static Singleton getInstance() {
      if (instance == null) {
          synchronized (Singleton.class) {
              if (instance == null) instance = new Singleton();
          }
      }
      return instance;
  }
  ```
- **Enum singleton** (Joshua Bloch's recommendation) — thread-safe, serialization-safe, reflection-safe:
  ```java
  public enum Singleton { INSTANCE; public void doStuff() {...} }
  ```

**Interview angles:**
- "Why is Singleton considered an anti-pattern by some?" → Global state, hard to unit-test (can't mock easily), hidden dependencies, problems with multiple classloaders.
- "How would you make it thread-safe?" → Walk through the variants above.
- "What about in a distributed system?" → Singleton is per-JVM. For cluster-wide uniqueness you need coordination (ZooKeeper, distributed locks).

---

## Simple Factory — Chapter 4 (technically not a "GoF pattern", but always taught)
**TL;DR:** Move object creation into one place — a factory class with a `create()` method.

**Problem:** `new ConcretePizza()` scattered through your code couples clients to concrete classes. If a new type is added, every caller must change.

**Structure:** A `SimplePizzaFactory` with `createPizza(String type)` returning a `Pizza`. Clients ask the factory; they never `new` directly.

**Interview angle:** Easy to confuse with Factory Method. *Simple Factory is not really a pattern — it's a programming idiom.* It still violates OCP because adding a new product means editing the factory's `if/switch`.

---

## Factory Method — Chapter 4
**TL;DR:** Define an interface for creating an object, but let subclasses decide which class to instantiate.

**Problem:** A framework knows *when* to create an object, but the application code knows *which* concrete class. Factory Method lets the framework call an abstract method that subclasses override.

**Structure:**
- `Creator` (abstract class) has an abstract `factoryMethod()` plus other methods that *use* what the factory method returns.
- `ConcreteCreator` subclasses override `factoryMethod()` to return concrete `Product` instances.

**The book's example:** `PizzaStore` defines `orderPizza()` (the algorithm — order/prep/bake/cut/box) and an abstract `createPizza()`. `NYPizzaStore` and `ChicagoPizzaStore` subclass and override `createPizza()` to return their regional variants.

**Real-world / Java:** `Collection.iterator()` is a factory method — every collection produces its own iterator type. So is `Calendar.getInstance()`.

**Interview angles:**
- "Difference between Factory Method and Abstract Factory?" → FM uses **inheritance** (subclass to vary the product); AF uses **composition** (you pass in a factory object that creates a family of products).
- "Why not just use Simple Factory?" → FM allows extension via subclassing (OCP-friendly); Simple Factory does not.

---

## Abstract Factory — Chapter 4
**TL;DR:** Provide an interface for creating **families** of related objects without specifying their concrete classes.

**Problem:** You need to create a *set* of products that work together — say, "New York" pizza ingredients (thin dough + marinara + reggiano) vs "Chicago" ingredients (thick dough + plum tomato + mozzarella). You want to guarantee consistency within a family.

**Structure:**
- `AbstractFactory` interface with methods like `createDough()`, `createSauce()`, `createCheese()`.
- `NYIngredientFactory`, `ChicagoIngredientFactory` implement them.
- Clients receive a factory and use it — they never know which concrete factory they got.

**Real-world / Java:** `javax.xml.parsers.DocumentBuilderFactory`, AWT's `Toolkit` (which creates platform-specific widgets — Windows/Mac/Linux variants).

**Interview angles:**
- "When to choose Factory Method vs Abstract Factory?" → One product → FM. Family of related products → AF.
- AF heavily uses DIP: client code depends only on the abstract factory + abstract products.

---

## Builder — Appendix
**TL;DR:** Construct a complex object step by step, separating construction from representation.

**Problem:** Constructors with 10 parameters become unreadable (`new Pizza(true, false, "thin", "marinara", null, 0, ...)`). Telescoping constructors (multiple overloads) get out of hand.

**Structure (modern fluent style):**
```java
Pizza p = new Pizza.Builder()
    .size("large")
    .crust("thin")
    .cheese("mozzarella")
    .addTopping("mushrooms")
    .build();
```

**Real-world / Java:** `StringBuilder`, `Stream.Builder`, Lombok's `@Builder`, Guava's `ImmutableList.builder()`, OkHttp `Request.Builder`.

**Interview angle:** Often the favorite answer to "how would you design a class with many optional parameters?" Pairs perfectly with **immutability** — build it up mutably, then freeze it on `.build()`.

---

## Prototype — Appendix
**TL;DR:** Create new objects by **cloning** an existing instance instead of instantiating from scratch.

**Problem:** Object creation is expensive (DB hit, heavy computation), or you don't know the concrete class at compile time, but you have an example instance lying around.

**Structure:** Objects implement a `clone()` method; clients ask the prototype to copy itself.

**Real-world / Java:** `Object.clone()` and `Cloneable`. Be very careful — `clone()` is shallow by default. For deep copies, override carefully or use serialization.

**Interview angle:** Rarely the right answer in modern Java (people prefer copy constructors or builders). Still worth knowing for the "clone is broken" discussion (effective Java has a whole item on this).

---

# Part 3 — Structural Patterns
*How to compose objects into larger structures while keeping them flexible.*

## Decorator — Chapter 3
**TL;DR:** Attach additional responsibilities to an object dynamically by wrapping it. A flexible alternative to subclassing for extending behavior.

**Problem:** Starbuzz wants to charge for coffee + condiments. Doing it with inheritance gives you a class explosion (`HouseBlendWithMilk`, `HouseBlendWithMilkAndMocha`, ...). You can't possibly subclass every combination.

**Structure:**
- A `Component` interface (`Beverage` with `cost()`).
- `ConcreteComponent` (`HouseBlend`, `Espresso`).
- `Decorator` (abstract) implements `Component` and **wraps** a `Component`.
- `ConcreteDecorator` (`Milk`, `Mocha`) adds its own cost on top of the wrapped beverage.

```java
Beverage b = new Mocha(new Milk(new HouseBlend()));
b.cost(); // walks the chain: HouseBlend.cost() + Milk + Mocha
```

**Real-world / Java:** `java.io` is *the* canonical example: `new BufferedReader(new InputStreamReader(new FileInputStream("f.txt")))`. Also `Collections.unmodifiableList(list)`, `Collections.synchronizedList(list)`.

**Interview angles:**
- Why this and not inheritance? → Combinatorial explosion + you can't add behavior at runtime with inheritance.
- Headlines the **Open-Closed Principle**: you can wrap with new decorators without modifying the base.
- Trade-off: lots of small objects, can be hard to debug; objects look the "same type" as their wrapped target which can confuse `instanceof` checks.

---

## Adapter — Chapter 7
**TL;DR:** Convert one interface into another the client expects. The "power-plug travel adapter" of OOP.

**Problem:** You have a `Duck` interface but a third-party `Turkey` class. You can't change either. You write a `TurkeyAdapter implements Duck` that holds a `Turkey` and translates `quack()` into `gobble()`.

**Two flavors:**
- **Object adapter** (composition) — adapter *holds* the adaptee. More flexible (Java's default since Java doesn't have multiple inheritance).
- **Class adapter** (multiple inheritance) — adapter *extends* both. Not possible in pure Java.

**Real-world / Java:** `Arrays.asList()` (adapts an array to a `List`), `Collections.enumeration()` (adapts `Iterator` to `Enumeration`), wrapping an `InputStream` in an `InputStreamReader`.

**Interview angle:** Comes up constantly in legacy / integration work — "I have a new system that needs to talk to an old library, how do I bolt them together without rewriting either?"

---

## Facade — Chapter 7
**TL;DR:** Provide a unified, simplified interface to a complex subsystem.

**Problem:** Watching a movie means: dim lights, drop screen, turn on projector, set input, switch amp, set volume, start DVD player... A `HomeTheaterFacade.watchMovie("Raiders")` hides all 12 steps.

**Difference from Adapter:** Adapter changes the *interface* of one class to match another. Facade simplifies the *use* of many classes.

**Real-world / Java:** `javax.faces.context.FacesContext`, SLF4J in front of multiple logging backends, JDBC's high-level API hiding driver internals. In microservices, an **API Gateway** is conceptually a facade.

**Interview angle:** Headlines the **Principle of Least Knowledge / Law of Demeter** — clients talk only to the facade, not to the 12 subsystem objects.

---

## Proxy — Chapter 11
**TL;DR:** Provide a surrogate or placeholder for another object to control access to it.

**Variants (all share the same structure):**
- **Remote Proxy** — represents an object in a different JVM. Local method calls forward over the network. Java RMI is the book's example.
- **Virtual Proxy** — lazy initialization. Holds a placeholder, only creates the expensive real object on first use (e.g., the album-cover viewer that doesn't fetch images until needed). Hibernate's lazy-loaded entities work this way.
- **Protection Proxy** — controls access based on caller's permissions. Spring Security method-level checks are this.
- **Caching Proxy** — caches results of expensive operations.
- **Smart Reference** — adds bookkeeping (reference counting, logging) on access.

**Real-world / Java:**
- `java.lang.reflect.Proxy` (Dynamic Proxy) — the book's matchmaking-service example.
- All AOP frameworks (Spring, AspectJ) generate proxies at runtime to weave in cross-cutting concerns (transactions, security, logging).
- ORM lazy-loading.

**Interview angles:**
- "How does Spring `@Transactional` actually work?" → Spring wraps your bean in a dynamic proxy that opens a transaction before delegating and commits/rolls back after.
- Proxy vs Decorator? → Same *structure* (wrapping). Different *intent*: decorator adds behavior; proxy controls access.

---

## Composite — Chapter 9
**TL;DR:** Treat individual objects and compositions of objects uniformly — both implement the same interface.

**Problem:** Menu items and submenus should both be "displayable". You want to treat a single item and a whole menu with the same code.

**Structure:**
- `Component` (`MenuComponent`) — common interface with operations like `print()`.
- `Leaf` (`MenuItem`) — no children.
- `Composite` (`Menu`) — holds a list of children and recursively delegates.

**Real-world / Java:** `java.awt.Container` (panel contains buttons + other panels), file systems (directories contain files + directories), HTML DOM, Spring's `Composite` `MessageSource`, JSON/XML trees.

**Interview angles:**
- The "safety vs transparency trade-off" — should leaves expose `add()/remove()` methods (transparency, runtime errors) or only composites (safety, clients need type checks)? GoF leans toward transparency; the book covers this debate.
- Recursion shows up naturally; expect a follow-up about traversal (often combined with **Iterator** or **Visitor**).

---

## Bridge — Appendix
**TL;DR:** Decouple an abstraction from its implementation so the two can vary independently.

**Problem:** You have `Shape` with subclasses `Circle`/`Square`, and you want to render them on different platforms (`OpenGL`/`DirectX`). Naïve inheritance gives you `OpenGLCircle`, `DirectXCircle`, `OpenGLSquare`, `DirectXSquare` — m × n explosion. Bridge says: have `Shape` hold a reference to a `Renderer`, and let each evolve separately. Now it's m + n.

**Real-world / Java:** JDBC (`Driver` is the implementation, `Connection`/`Statement` is the abstraction), AWT's component/peer split.

**Interview angle:** "When you see two orthogonal hierarchies, think Bridge." Often confused with Strategy — structurally similar, but Bridge is about a long-lived structural relationship, Strategy is about swapping algorithms at runtime.

---

## Flyweight — Appendix
**TL;DR:** Share fine-grained objects efficiently by separating intrinsic (shared) and extrinsic (per-context) state.

**Problem:** A forest with 1,000,000 trees. Each `Tree` storing its sprite, mesh, and texture wastes memory. Store those once in a `TreeType` (the flyweight); each `Tree` stores only its `(x, y)` and a reference to its `TreeType`.

**Real-world / Java:** `Integer.valueOf()` cache for -128 to 127, the String constant pool, `Character.valueOf()`. Game engines for tile / particle systems.

**Interview angle:** Memory optimization. The trade-off: extra complexity, and the shared state must be immutable.

---

# Part 4 — Behavioral Patterns
*How objects communicate and split responsibilities.*

## Strategy — Chapter 1 (the book opens with it)
**TL;DR:** Define a family of algorithms, encapsulate each, and make them interchangeable. The algorithm varies independently from the clients that use it.

**Problem:** The `Duck` example. Some ducks fly, some don't, some quack, some squeak, some are silent. Putting behavior into `Duck` with inheritance fails the moment a subclass needs different behavior (rubber duck shouldn't fly).

**Structure:**
- `Duck` *has-a* `FlyBehavior` and *has-a* `QuackBehavior` (composition!).
- Behaviors are interfaces with multiple implementations.
- `setFlyBehavior(FlyWithWings)` swaps at runtime.

**Real-world / Java:** `Comparator` passed to `Collections.sort()`, `Runnable` passed to `Thread`, payment-method strategies in checkout, compression-algorithm strategies. Anywhere you pass a lambda, you're using Strategy.

**Interview angle:** Strategy is *the* go-to answer for "how to avoid `if/else` chains based on type." Headlines **composition over inheritance** and **encapsulate what varies**.

---

## Observer — Chapter 2
**TL;DR:** Define a one-to-many dependency between objects so that when one (the **subject**) changes state, all its dependents (**observers**) are notified automatically.

**Problem:** A weather station has temperature/humidity/pressure data, and multiple displays want to show it. Hardcoding the displays into the station is rigid; observer makes it pluggable.

**Structure:**
- `Subject` interface: `registerObserver()`, `removeObserver()`, `notifyObservers()`.
- `Observer` interface: `update()`.
- Subject keeps a list of observers and calls `update()` on each when state changes.

**Push vs pull:** Push — subject sends new data in `update(data)`. Pull — subject just signals "I changed", observers ask back for what they want. Pull is more flexible.

**Real-world / Java:**
- GUI event listeners (every `addActionListener()` in Swing).
- `java.util.Observer` / `Observable` (deprecated since Java 9 — use `PropertyChangeListener` or reactive libs instead).
- Reactive Streams (`Flow.Publisher`/`Subscriber` in `java.util.concurrent`).
- Pub/sub systems at scale (Kafka, RabbitMQ) — distributed Observer.

**Interview angles:**
- "Difference between Observer and pub/sub?" → Observer is in-process (subject knows observers). Pub/sub adds a broker that decouples publishers from subscribers — they don't know each other at all.
- Headlines **loose coupling** — subject only knows the `Observer` interface.
- Watch for memory leaks: forgotten observers keep subjects alive (very common in Android/Swing).

---

## Command — Chapter 6
**TL;DR:** Encapsulate a request as an object, so you can parameterize clients, queue or log requests, and support undo.

**Problem:** A universal remote control has slots that should work for *any* device — light, ceiling fan, stereo. The remote shouldn't know how each device works.

**Structure:**
- `Command` interface: `execute()` (and often `undo()`).
- `ConcreteCommand` (`LightOnCommand`) holds a reference to a `Receiver` (`Light`) and calls `light.on()` in `execute()`.
- `Invoker` (the remote) holds commands and calls `execute()`.
- `Client` configures everything: creates the command, sets the receiver, hands it to the invoker.

**Extensions:**
- **Undo** — each command remembers the prior state.
- **Macro command** — a command that holds a list of commands.
- **Queueing** — thread pools take `Runnable` (a degenerate command) off a queue.
- **Logging / replay** — write executed commands to a log; on crash, re-execute from the log to reach the prior state (basically the write-ahead log idea in databases).

**Real-world / Java:** `Runnable`, `Callable`, `ActionListener` in Swing, transaction logs in DBs, every "redo" button in every editor ever made.

**Interview angles:**
- "How would you implement undo in a text editor?" → Command pattern with a stack of executed commands.
- "How does a task queue work?" → Each task is a command object submitted to an `ExecutorService`.

---

## Template Method — Chapter 8
**TL;DR:** Define the **skeleton** of an algorithm in a base class, deferring some steps to subclasses. Subclasses change parts without changing the algorithm's structure.

**Problem:** Coffee and tea both follow the same recipe (boil water → brew → pour → add condiments), but the brew step and condiments differ.

**Structure:**
- Abstract class with a `final` `templateMethod()` that calls a sequence of methods, some `abstract` (subclasses must implement), some with default impls (subclasses can override), some `final` (subclasses can't).
- **Hooks** — optional methods with default no-op or default-true behavior; subclasses override only if they need to.

**Real-world / Java:** `java.util.AbstractList`, `HttpServlet.service()` (calls `doGet`/`doPost` you override), Spring's `JdbcTemplate`, JUnit's `@Before`/`@Test`/`@After` lifecycle.

**Interview angles:**
- Headlines the **Hollywood Principle**: "Don't call us, we'll call you." The framework controls the flow; your subclass plugs into specific extension points.
- Difference from Strategy? → Strategy uses *composition* (swap whole algorithm); Template Method uses *inheritance* (subclass to vary parts).

---

## Iterator — Chapter 9
**TL;DR:** Provide a way to access the elements of an aggregate object sequentially without exposing its underlying representation.

**Problem:** The Diner uses an `ArrayList`, the Pancake House uses an array. The waitress shouldn't have to know — she just wants to walk through each menu's items.

**Structure:** `Iterator` interface with `hasNext()` and `next()`. Each aggregate exposes a `createIterator()` method returning its own iterator.

**Real-world / Java:** `java.util.Iterator`, `Iterable`, the enhanced `for` loop (`for (X x : collection)`) — that loop calls `iterator()` under the hood.

**Interview angles:**
- Headlines the **Single Responsibility Principle** — the collection holds elements; the iterator handles traversal. Two different reasons to change → two classes.
- Internal vs external iterator? External (`Iterator.next()`) — client controls. Internal (`stream.forEach(...)`) — iterator controls, you supply the operation. Java 8+ pushed hard toward internal.

---

## State — Chapter 10
**TL;DR:** Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.

**Problem:** The gumball machine has states: `NoQuarter`, `HasQuarter`, `Sold`, `SoldOut`. Coding this with `if/else` on a state field becomes a tangled mess as states multiply.

**Structure:**
- A `State` interface with methods for every event (`insertQuarter`, `ejectQuarter`, `turnCrank`, `dispense`).
- A concrete state class for each state, implementing only the relevant transitions.
- The `Context` (gumball machine) holds a current `State` and delegates events to it: `currentState.insertQuarter()`. States call back to `context.setState(...)` to transition.

**Difference from Strategy:** Identical class diagram, different intent. Strategy: client picks the algorithm and it stays put. State: the context transitions between states based on internal logic.

**Real-world / Java:** TCP connection states, order workflows (Pending → Paid → Shipped → Delivered), parser/lexer states, game character states (idle/walking/running/jumping).

**Interview angles:**
- "How would you model a vending machine / ATM / order lifecycle?" → State pattern is the textbook answer.
- For large state machines, consider Spring Statemachine or AWS Step Functions instead of hand-rolling.

---

## Chain of Responsibility — Appendix
**TL;DR:** Pass a request along a chain of handlers; each handler either processes it or forwards it.

**Problem:** Logging filters with multiple levels, expense approval workflows (manager → director → VP based on amount), HTTP middleware pipelines.

**Real-world / Java:** `javax.servlet.Filter` chains, Spring Security's filter chain, Netty's `ChannelPipeline`, every web framework's middleware (Express, ASP.NET).

**Interview angle:** Crucial for any system-design question involving request processing pipelines, especially API gateways and middleware.

---

## Mediator — Appendix
**TL;DR:** Define an object that encapsulates how a set of objects interact. Promotes loose coupling by keeping objects from referring to each other directly.

**Problem:** A dialog box has 10 widgets that all affect each other ("if checkbox A is on, disable text field B, hide button C..."). Hard-wiring every widget to every other is N² edges. A mediator (the dialog) sits in the middle: widgets only talk to the mediator.

**Real-world:** Air traffic control (planes don't talk to each other, only to the tower), chat rooms (clients talk to the server, not to each other directly), Redux store in front-end apps.

**Interview angle:** Often confused with Facade. Facade simplifies access to a subsystem (one-way). Mediator coordinates peer-to-peer interactions (multi-way).

---

## Memento — Appendix
**TL;DR:** Capture an object's internal state without violating encapsulation so the object can be restored later.

**Real-world:** Undo stacks, save game files, database snapshots, "revert to" features in editors.

**Interview angle:** Often paired with Command (commands hold mementos to undo).

---

## Visitor — Appendix
**TL;DR:** Represent an operation to be performed on the elements of an object structure. Lets you define a new operation without changing the classes of the elements.

**Problem:** You have a stable hierarchy of types (AST nodes: `IfNode`, `WhileNode`, `BinaryOpNode`) and many operations (`print`, `optimize`, `compile`, `type-check`) you want to add. Adding each operation as a method on every node bloats the hierarchy. Visitor inverts: each operation is a `Visitor` with a `visit(IfNode)`, `visit(WhileNode)`, etc. Nodes implement `accept(Visitor v)` which calls `v.visit(this)` — the double dispatch.

**Real-world / Java:** AST traversal in compilers (`javac`, ANTLR), Java's annotation processors, file-tree walkers (`Files.walkFileTree` takes a `FileVisitor`).

**Interview angles:**
- Trade-off: easy to add **operations**, hard to add **types** (every visitor must add a new `visit` method). Composite's trade-off is the opposite.
- Showcases double dispatch — useful concept on its own.

---

## Interpreter — Appendix
**TL;DR:** Given a language, define a representation for its grammar along with an interpreter that uses the representation to interpret sentences in the language.

**Real-world:** Regex engines, SQL parsers, expression evaluators, configuration DSLs.

**Interview angle:** Rare in modern practice — most people use parser generators (ANTLR) or write hand-coded recursive descent. Mostly worth knowing it exists.

---

# Part 5 — Compound Patterns

## Model-View-Controller (MVC) — Chapter 12
**TL;DR:** Separate the data (Model), the UI (View), and the input-handling (Controller). The view observes the model; the controller mediates user input.

**Patterns inside MVC:**
- **Observer** — view observes the model and re-renders on change.
- **Strategy** — controller is a strategy for handling input; you can swap controllers.
- **Composite** — UI widgets in the view typically form a composite tree.

**Modern descendants:**
- **MVP** (Model-View-Presenter) — presenter holds presentation logic, view is dumb.
- **MVVM** (Model-View-ViewModel) — view-model exposes observable bindings; popular in Angular, Vue, WPF.
- **Flux/Redux** — unidirectional data flow; closer to event sourcing + observer.

**Interview angle:** Almost every system-design discussion of UI architecture eventually touches MVC. Be ready to articulate why decoupling rendering from data from input handling matters (testability, multiple views of the same data, swappable UIs).

---

# Part 6 — Patterns in the Real World (Chapter 13)

The book closes with three big themes:

1. **Pattern catalogs are organized by intent.** GoF's three categories (Creational, Structural, Behavioral) are the canon. Newer catalogs add Concurrency patterns (Reactor, Active Object, Producer-Consumer) and Architectural patterns (Layers, Pipes-and-Filters, Microservices).

2. **Anti-patterns matter as much as patterns.** Common ones:
   - **God Object** — one class does everything.
   - **Spaghetti Code** — no structure.
   - **Golden Hammer** — applying one pattern to everything ("when you have a hammer...").
   - **Premature Optimization / Patternitis** — using patterns for their own sake. *Patterns add complexity; only use them when the complexity is justified by real flexibility you need.*

3. **The shared vocabulary is the real win.** If you say "we should use a Strategy here", a teammate immediately knows the structure, the trade-offs, and the alternatives. That's what interviewers are testing when they ask about patterns.

---

# Part 7 — Interview Cheat Sheet

### How interview questions usually frame patterns

| Question style | What they want to hear |
|---|---|
| "Design X" (parking lot, vending machine, chess) | Pattern names: Strategy for movement, State for order lifecycle, Factory for piece creation, Composite for board hierarchy. |
| "Refactor this code that has 5 `if/else` branches by type" | Strategy or State. |
| "How do you decouple this notification system?" | Observer / Pub-Sub. |
| "How do you build undo?" | Command + Memento. |
| "How do you wrap a legacy API?" | Adapter (or Facade if the API is messy AND large). |
| "How do you add features without modifying existing classes?" | Decorator (per-instance) or Strategy (per-algorithm). OCP. |
| "How do you avoid instantiating expensive objects?" | Virtual Proxy or Flyweight. |
| "How does Spring `@Transactional` / AOP work?" | Dynamic Proxy. |

### Easily-confused pairs (these come up *constantly*)

- **Strategy vs State** — same diagram. Strategy: client picks. State: context transitions itself.
- **Adapter vs Facade** — both wrap. Adapter changes one interface; Facade simplifies many.
- **Decorator vs Proxy** — both wrap. Decorator adds behavior; Proxy controls access.
- **Decorator vs Strategy** — Decorator wraps the whole object; Strategy swaps one piece of behavior inside the object.
- **Factory Method vs Abstract Factory** — FM creates one product via subclassing; AF creates a family via composition.
- **Bridge vs Strategy** — same diagram. Bridge: structural, long-lived. Strategy: behavioral, hot-swappable.
- **Mediator vs Facade** — Mediator coordinates peer interactions (multi-way). Facade simplifies access (one-way).
- **Template Method vs Strategy** — both vary an algorithm. TM uses inheritance (compile-time); Strategy uses composition (runtime).

### The 30-second pitch for each (memorize these)

- **Singleton** — one instance, global access.
- **Factory Method** — defer instantiation to subclasses.
- **Abstract Factory** — create families of related objects.
- **Builder** — construct complex objects step by step.
- **Prototype** — clone existing instances.
- **Adapter** — convert one interface to another.
- **Bridge** — decouple abstraction from implementation.
- **Composite** — treat individuals and groups uniformly.
- **Decorator** — wrap to add behavior dynamically.
- **Facade** — simplified interface to a complex subsystem.
- **Flyweight** — share fine-grained objects.
- **Proxy** — surrogate that controls access.
- **Chain of Responsibility** — pass request along a chain until handled.
- **Command** — encapsulate a request as an object.
- **Iterator** — sequential access without exposing internals.
- **Mediator** — centralize peer interactions.
- **Memento** — capture and restore state.
- **Observer** — one-to-many notification of state changes.
- **State** — change behavior with state, looks like changing class.
- **Strategy** — interchangeable algorithms.
- **Template Method** — algorithm skeleton, override steps.
- **Visitor** — add operations to a structure without modifying it.

---

# Part 8 — Suggested Reading Order

If you have **2 days** before an interview, study in this order:
1. **Principles** (Part 1) — non-negotiable foundation.
2. **Strategy, Observer, Decorator, Factory Method, Singleton, Command, State, Adapter, Facade, Template Method, Iterator** — the "core 11" that come up most often.
3. **Proxy, Composite, Builder, Abstract Factory** — second tier, still very common.
4. **Cheat sheet** (Part 7) — drill the easily-confused pairs.

If you have **a week**, also cover Bridge, Flyweight, Chain of Responsibility, Visitor, Mediator, Memento, Prototype, Interpreter — and re-read the book's Chapter 12 (MVC compound pattern) plus Chapter 13 (anti-patterns).

For deeper study after the basics, the natural next book is the **Gang of Four**'s *Design Patterns: Elements of Reusable Object-Oriented Software* — denser, more formal, and what the interviewer probably read.

Good luck.
