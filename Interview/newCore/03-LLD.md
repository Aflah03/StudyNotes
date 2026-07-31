# LLD (Low-Level Design) — Interview Prep Guide

> Goal: In an LLD / machine-coding round you're handed a fuzzy problem ("Design a parking lot") and 45–90 minutes. Success = clarify requirements → identify entities → apply the right patterns → write clean, extensible OO code. This guide gives you the **framework**, the **patterns**, and **fully worked problems**.

---

## 0. What LLD interviews actually measure

Not "can you finish." They measure:
- **Requirement clarification** — do you ask before coding?
- **Object modeling** — sensible classes, right relationships, good encapsulation.
- **Extensibility** — can new requirements be added without rewriting? (SOLID, patterns.)
- **Clean code** — readable names, no god classes, separation of concerns.

A half-finished but clean, extensible design beats a fully-working spaghetti mess.

---

## 1. The universal 6-step framework (use this every time)

**Step 1 — Clarify & scope (2–5 min).** Ask questions. State assumptions out loud.
> "Design a parking lot" → How many floors? Vehicle types (car/bike/truck)? Payment (hourly/flat)? Multiple entry/exit gates? Do we assign specific spots or just count? I'll assume multi-floor, three vehicle types, hourly pricing, multiple gates.

**Step 2 — Identify entities (nouns).** Extract the core objects from the problem. Parking lot → `ParkingLot`, `Floor`, `ParkingSpot`, `Vehicle`, `Ticket`, `Gate`, `Payment`.

**Step 3 — Define relationships.** ParkingLot *has* Floors (composition). Floor *has* Spots. Ticket *references* Vehicle + Spot. Draw it or describe it.

**Step 4 — Define the APIs / behaviors (verbs).** `parkVehicle()`, `unpark()`, `findAvailableSpot()`, `calculateFee()`, `processPayment()`.

**Step 5 — Apply patterns where they earn their place.** Pricing that varies → Strategy. Creating spot types → Factory. Single lot instance → Singleton. Notifying gates when full → Observer. *Don't force patterns — use them when they solve a real variation point.*

**Step 6 — Code the core, mention extensions.** Write the key classes cleanly. Verbally note what you'd add with more time (persistence, concurrency locks, edge cases).

**Say this early:** "Let me clarify a few requirements first, then lay out the entities, then code the core flow." Interviewers relax immediately when you have a process.

---

## 2. SOLID — the backbone of every good LLD (with code)

### S — Single Responsibility Principle
> A class should have exactly one reason to change.

```java
// ❌ God class: 3 responsibilities, 3 reasons to change
class Invoice {
    void calculateTotal() {}
    void printInvoice() {}       // printing logic
    void saveToDatabase() {}     // persistence logic
}

// ✅ Split
class Invoice { void calculateTotal() {} }
class InvoicePrinter { void print(Invoice i) {} }
class InvoiceRepository { void save(Invoice i) {} }
```
*Benefit:* changing how you print doesn't risk breaking calculation.

### O — Open/Closed Principle
> Open for extension, closed for modification.

```java
// ❌ Every new shape forces editing this method
double area(Shape s) {
    if (s.type == CIRCLE) return 3.14 * s.r * s.r;
    else if (s.type == SQUARE) return s.side * s.side;   // and again... and again...
}

// ✅ Add a subclass, touch nothing existing
interface Shape { double area(); }
class Circle implements Shape { double r; public double area() { return 3.14*r*r; } }
class Square implements Shape { double side; public double area() { return side*side; } }
```

### L — Liskov Substitution Principle
> Subtypes must be substitutable for their base type without breaking behavior.

```java
// ❌ Classic violation
class Rectangle { void setWidth(int w){} void setHeight(int h){} }
class Square extends Rectangle {
    void setWidth(int w) { this.w = w; this.h = w; }  // side effect!
}
// Code expecting Rectangle: setWidth(5); setHeight(4); assert area==20 → FAILS for Square (area 16)
// Fix: don't model Square as a subtype of Rectangle. Use a common Shape abstraction.
```

### I — Interface Segregation Principle
> Don't force a class to depend on methods it doesn't use.

```java
// ❌ Fat interface
interface Worker { void work(); void eat(); }
class Robot implements Worker { void work(){} void eat(){} }  // robots don't eat!

// ✅ Segregate
interface Workable { void work(); }
interface Eatable { void eat(); }
class Robot implements Workable { public void work(){} }
class Human implements Workable, Eatable { public void work(){} public void eat(){} }
```

### D — Dependency Inversion Principle
> High-level modules depend on abstractions, not concrete low-level modules.

```java
// ❌ Tightly coupled to a concrete sender
class NotificationService {
    private EmailSender sender = new EmailSender();  // hard-wired
}

// ✅ Depend on abstraction, inject the concrete
interface MessageSender { void send(String msg); }
class EmailSender implements MessageSender { public void send(String m){} }
class SmsSender implements MessageSender { public void send(String m){} }

class NotificationService {
    private MessageSender sender;
    NotificationService(MessageSender sender) { this.sender = sender; }  // injected
}
```

---

## 3. Design Patterns you MUST know (with code + when to use)

Interviewers expect these to come up naturally as you solve problems. Learn the *trigger* ("when do I reach for this?") more than the definition.

### Creational

**Singleton** — exactly one instance, global access.
*When:* config, logger, connection pool, the single `ParkingLot` object.
```java
class Logger {
    private static volatile Logger instance;
    private Logger() {}                          // private ctor blocks `new`
    static Logger getInstance() {
        if (instance == null) {                  // double-checked locking (thread-safe)
            synchronized (Logger.class) {
                if (instance == null) instance = new Logger();
            }
        }
        return instance;
    }
}
```
*Trade-off / follow-up:* Singletons are effectively global state → harder to test, can hide dependencies. In production, prefer dependency injection over a hand-rolled singleton where possible.

**Factory Method** — a method decides which concrete class to instantiate; caller depends only on the abstraction.
*When:* object creation logic varies or you want to hide `new`. E.g., create the right `ParkingSpot` for a vehicle type.
```java
class VehicleFactory {
    static Vehicle create(String type) {
        switch (type) {
            case "car":  return new Car();
            case "bike": return new Bike();
            default: throw new IllegalArgumentException(type);
        }
    }
}
```

**Abstract Factory** — a factory of factories; creates *families* of related objects (e.g., a UI toolkit that makes matching Windows or Mac buttons + checkboxes).

**Builder** — construct a complex object step by step; avoids telescoping constructors.
*When:* an object with many optional fields. E.g., building an HTTP request, a `Pizza`, a `User` with 10 optional attributes.
```java
Pizza p = new Pizza.Builder()
    .size("large")
    .addTopping("mushroom")
    .addTopping("olive")
    .cheeseBurst(true)
    .build();
```

### Structural

**Adapter** — make two incompatible interfaces work together (a wrapper/translator).
*When:* integrating a third-party library whose interface differs from yours. E.g., adapting a legacy `XmlPaymentGateway` to your `PaymentProcessor` interface.

**Decorator** — add behavior to an object dynamically by wrapping it, without changing its class.
*When:* layered optional features. Classic: coffee + milk + sugar; or Java's `BufferedReader(new FileReader(...))`.
```java
Coffee c = new Milk(new Sugar(new SimpleCoffee()));  // each wraps and adds cost
c.cost();  // base + sugar + milk
```

**Facade** — a simplified front-end over a complex subsystem.
*When:* hide a messy multi-step process behind one call. E.g., `videoConverter.convert(file, "mp4")` hiding codecs, bitrate, audio mixing.

**Proxy** — a stand-in that controls access to another object (lazy loading, access control, caching).
*When:* expensive object you want to load on demand, or add an access-check layer.

### Behavioral

**Strategy** — encapsulate interchangeable algorithms behind a common interface; swap at runtime. **The most useful LLD pattern.**
*When:* one behavior has several variants. E.g., pricing strategies, sorting strategies, payment methods, route-finding algorithms.
```java
interface PricingStrategy { double price(long minutes); }
class HourlyPricing implements PricingStrategy { public double price(long m){ return Math.ceil(m/60.0)*20; } }
class FlatPricing   implements PricingStrategy { public double price(long m){ return 100; } }

class Ticket {
    private PricingStrategy strategy;
    Ticket(PricingStrategy s) { this.strategy = s; }   // inject strategy
    double getBill(long minutes) { return strategy.price(minutes); }
}
```

**Observer** — when one object changes state, all its dependents are notified automatically (publish/subscribe).
*When:* event notification. E.g., stock price change → update all watching displays; parking-lot-full → notify all entry gates; YouTube channel → notify subscribers.
```java
interface Observer { void update(String event); }
class Subject {
    private List<Observer> observers = new ArrayList<>();
    void subscribe(Observer o) { observers.add(o); }
    void notifyAll(String event) { for (Observer o : observers) o.update(event); }
}
```

**State** — an object alters its behavior when its internal state changes; each state is a class.
*When:* an entity with distinct modes and transitions. E.g., a vending machine (`NoCoin`, `HasCoin`, `Dispensing`), an order (`Placed`→`Shipped`→`Delivered`), a traffic light.

**Command** — encapsulate a request as an object (enables undo/redo, queuing, logging).
*When:* undo/redo, task queues, remote-control buttons.

**Chain of Responsibility** — pass a request along a chain of handlers until one handles it.
*When:* middleware pipelines, approval workflows, logging levels, ATM cash dispensing by denomination.

---

## 4. UML quick reference (enough to draw on a whiteboard)

**Class box:** three compartments — name / attributes / methods. `+` public, `-` private, `#` protected.

**Relationship arrows:**
- **Inheritance** (is-a): solid line, hollow triangle arrow → points to parent.
- **Realization** (implements interface): dashed line, hollow triangle.
- **Association** (uses-a): plain solid line.
- **Aggregation** (has-a, independent lifecycle): hollow diamond at the "whole" end.
- **Composition** (has-a, dependent lifecycle): filled diamond at the "whole" end.
- **Dependency** (temporary use, e.g., a param): dashed arrow.

**Multiplicity:** `1`, `0..1`, `*` (many), `1..*` (one or more) — written at the line ends. E.g., `ParkingLot 1 — 1..* Floor`.

---

## 5. WORKED PROBLEM 1 — Parking Lot (the "hello world" of LLD)

### Requirements (after clarifying)
Multi-floor lot; vehicle types Car/Bike/Truck; each floor has spots sized for vehicle types; hourly pricing; ticket issued on entry, fee paid on exit; multiple gates.

### Entities & relationships
- `ParkingLot` (Singleton) *has* many `Floor` (composition)
- `Floor` *has* many `ParkingSpot`
- `ParkingSpot` has a `SpotType` (COMPACT/LARGE/BIKE) and holds a `Vehicle`
- `Vehicle` (abstract) → `Car`, `Bike`, `Truck`
- `Ticket` references a `Vehicle`, a `ParkingSpot`, entry time
- `PricingStrategy` (Strategy) computes fee
- `ParkingLot` uses a spot-assignment step (could be Strategy: nearest/random)

### Core code
```java
enum VehicleType { BIKE, CAR, TRUCK }
enum SpotType    { BIKE, COMPACT, LARGE }

abstract class Vehicle {
    private final String plate;
    private final VehicleType type;
    Vehicle(String plate, VehicleType type) { this.plate = plate; this.type = type; }
    VehicleType getType() { return type; }
    String getPlate() { return plate; }
}
class Car   extends Vehicle { Car(String p)   { super(p, VehicleType.CAR); } }
class Bike  extends Vehicle { Bike(String p)  { super(p, VehicleType.BIKE); } }
class Truck extends Vehicle { Truck(String p) { super(p, VehicleType.TRUCK); } }

class ParkingSpot {
    private final String id;
    private final SpotType type;
    private Vehicle vehicle;                 // null = free
    ParkingSpot(String id, SpotType type) { this.id = id; this.type = type; }
    boolean isFree() { return vehicle == null; }
    boolean canFit(Vehicle v) {              // fit rules
        if (type == SpotType.LARGE) return true;
        if (type == SpotType.COMPACT) return v.getType() != VehicleType.TRUCK;
        return v.getType() == VehicleType.BIKE;  // BIKE spot
    }
    void park(Vehicle v) { this.vehicle = v; }
    void vacate() { this.vehicle = null; }
    String getId() { return id; }
}

class Floor {
    private final int number;
    private final List<ParkingSpot> spots;
    Floor(int number, List<ParkingSpot> spots) { this.number = number; this.spots = spots; }
    Optional<ParkingSpot> findSpot(Vehicle v) {
        return spots.stream().filter(s -> s.isFree() && s.canFit(v)).findFirst();
    }
}

// Strategy: pricing can vary without changing Ticket
interface PricingStrategy { double calculate(long minutes, VehicleType type); }
class HourlyPricing implements PricingStrategy {
    public double calculate(long minutes, VehicleType type) {
        long hours = (long) Math.ceil(minutes / 60.0);
        int rate = switch (type) { case BIKE -> 10; case CAR -> 20; case TRUCK -> 40; };
        return hours * rate;
    }
}

class Ticket {
    private final String id;
    private final Vehicle vehicle;
    private final ParkingSpot spot;
    private final long entryTime;
    Ticket(String id, Vehicle v, ParkingSpot s) {
        this.id = id; this.vehicle = v; this.spot = s; this.entryTime = System.currentTimeMillis();
    }
    long minutesParked() { return (System.currentTimeMillis() - entryTime) / 60000; }
    Vehicle getVehicle() { return vehicle; }
    ParkingSpot getSpot() { return spot; }
}

class ParkingLot {                            // Singleton
    private static final ParkingLot INSTANCE = new ParkingLot();
    private List<Floor> floors = new ArrayList<>();
    private PricingStrategy pricing = new HourlyPricing();
    private ParkingLot() {}
    static ParkingLot getInstance() { return INSTANCE; }
    void addFloor(Floor f) { floors.add(f); }

    Ticket park(Vehicle v) {
        for (Floor f : floors) {
            Optional<ParkingSpot> spot = f.findSpot(v);
            if (spot.isPresent()) {
                spot.get().park(v);
                return new Ticket(UUID.randomUUID().toString(), v, spot.get());
            }
        }
        throw new IllegalStateException("Parking full for " + v.getType());
    }

    double unpark(Ticket t) {
        double fee = pricing.calculate(t.minutesParked(), t.getVehicle().getType());
        t.getSpot().vacate();
        return fee;
    }
}
```
**Extensions to mention:** concurrency (lock a spot during assignment so two gates don't grab the same one), spot-assignment Strategy (nearest to entrance), Observer to signal "floor full" to gates, persistence layer, reservation support.

---

## 6. WORKED PROBLEM 2 — Vending Machine (showcases the State pattern)

### Why State
A vending machine's behavior *depends on its current mode*: inserting a coin means different things when idle vs when already dispensing. Big `if/else` on a status flag gets ugly; the **State pattern** makes each mode a class with clean transitions.

```java
interface State {
    void insertCoin(VendingMachine m, int amount);
    void selectItem(VendingMachine m, String item);
    void dispense(VendingMachine m);
}

class VendingMachine {
    private State state;
    private int balance = 0;
    private Map<String, Integer> inventory;  // item -> count
    private Map<String, Integer> prices;

    VendingMachine() { this.state = new IdleState(); }
    void setState(State s) { this.state = s; }
    void addBalance(int a) { balance += a; }
    int getBalance() { return balance; }
    // delegate everything to the current state
    void insertCoin(int a) { state.insertCoin(this, a); }
    void selectItem(String i) { state.selectItem(this, i); }
    void dispense() { state.dispense(this); }
    // ... inventory/price accessors
}

class IdleState implements State {
    public void insertCoin(VendingMachine m, int amount) {
        m.addBalance(amount);
        m.setState(new HasMoneyState());       // transition
    }
    public void selectItem(VendingMachine m, String i) { System.out.println("Insert coin first"); }
    public void dispense(VendingMachine m) { System.out.println("Insert coin first"); }
}

class HasMoneyState implements State {
    public void insertCoin(VendingMachine m, int amount) { m.addBalance(amount); }
    public void selectItem(VendingMachine m, String item) {
        // check price & stock, then transition to DispensingState
        m.setState(new DispensingState(item));
    }
    public void dispense(VendingMachine m) { System.out.println("Select an item first"); }
}
// DispensingState: reduce stock, return change, reset to IdleState...
```
**Talking point:** "Adding a new mode (e.g., `MaintenanceState`) means adding a class, not editing a giant switch — that's Open/Closed via the State pattern."

---

## 7. WORKED PROBLEM 3 — Splitwise / expense sharing (data modeling + strategy)

### Requirements
Users create groups, add expenses; an expense is split **equally**, by **exact amount**, or by **percentage**; track who owes whom; simplify debts.

### Key design
- `User`, `Group (has users)`, `Expense`.
- `SplitStrategy` (Strategy pattern): `EqualSplit`, `ExactSplit`, `PercentSplit` — each turns a total into per-user shares.
- A `BalanceSheet` (map of `userA -> userB -> amount`) updated after each expense.

```java
interface SplitStrategy {
    Map<User, Double> split(double amount, List<User> participants, List<Double> values);
}
class EqualSplit implements SplitStrategy {
    public Map<User, Double> split(double amount, List<User> p, List<Double> v) {
        double share = amount / p.size();
        Map<User, Double> res = new HashMap<>();
        for (User u : p) res.put(u, share);
        return res;
    }
}

class ExpenseManager {
    private Map<User, Map<User, Double>> balances = new HashMap<>();   // owed[a][b] = a owes b
    void addExpense(User paidBy, double amount, List<User> participants,
                    SplitStrategy strategy, List<Double> values) {
        Map<User, Double> shares = strategy.split(amount, participants, values);
        for (Map.Entry<User, Double> e : shares.entrySet()) {
            User u = e.getKey();
            if (u.equals(paidBy)) continue;
            balances.computeIfAbsent(u, k -> new HashMap<>())
                    .merge(paidBy, e.getValue(), Double::sum);   // u now owes paidBy more
        }
    }
}
```
**Extension:** debt simplification (net out mutual debts and minimize transactions — a nice algorithm follow-up).

---

## 8. Other classic LLD problems (know the entities + patterns to reach for)

| Problem | Core entities | Patterns that fit |
|---------|--------------|-------------------|
| **Elevator system** | Elevator, Request, Controller, Direction | State (moving/idle), Strategy (scheduling: SCAN/LOOK), Observer |
| **BookMyShow / ticket booking** | Movie, Show, Screen, Seat, Booking, Payment | Strategy (pricing), **locking/concurrency** for seat hold, State (booking status) |
| **Tic-Tac-Toe / Chess** | Board, Cell, Player, Piece, Move, Game | Strategy (Piece move rules), State (game status), Factory (piece creation) |
| **Logging framework** | Logger, LogLevel, Appender (console/file), Formatter | Chain of Responsibility (levels), Singleton, Strategy (format) |
| **Rate limiter** | Request, Rule, Bucket/Window | Strategy (token bucket / sliding window), Singleton |
| **Notification system** | Notification, Channel (email/SMS/push), User | Strategy/Factory (channel), Observer, Decorator |
| **Snake & Ladder** | Board, Snake, Ladder, Dice, Player | Simple OO; Strategy (dice) |
| **Cache (LRU)** | Cache, Node, capacity | HashMap + doubly-linked list; Strategy for eviction policy |
| **Food delivery (Swiggy)** | User, Restaurant, Menu, Order, DeliveryAgent | Strategy (assignment, pricing), State (order lifecycle), Observer |

**LRU cache is a favorite** — often it's half data-structure, half design:
```java
// HashMap for O(1) lookup + doubly-linked list for O(1) recency updates
class LRUCache {
    private final int capacity;
    private final Map<Integer, Node> map = new HashMap<>();
    private final Node head, tail;   // dummy head/tail
    // get(k): move node to front; put(k,v): insert front, evict tail if over capacity
}
```

---

## 9. Concurrency in LLD (the follow-up that separates seniors)

Many LLD problems have a shared-resource race: two people booking the same seat, two cars taking the same spot. Be ready to say:
- **The problem:** check-then-act race — both threads see "available", both proceed.
- **The fix:** guard the critical section. Options: `synchronized`/locks (`ReentrantLock`), atomic compare-and-set, or a DB-level optimistic lock (version column) / `SELECT ... FOR UPDATE`.
- **Seat booking specifically:** put a *temporary hold* (with TTL) on the seat during payment; release if payment times out. This is both a concurrency and a State discussion.

---

## 10. Clean-code checklist the interviewer is silently ticking

- Meaningful names (`findAvailableSpot`, not `f1`).
- No god classes — each class has one responsibility.
- Program to interfaces, inject dependencies.
- Enums for fixed sets (vehicle types, states) — not magic strings.
- Encapsulate: private fields, behavior on the object that owns the data.
- No premature patterns — introduce a pattern only when there's a real variation point.
- Handle the obvious edge cases or at least *name* them ("full lot", "invalid ticket").

---

## 11. Practice problems (build these end-to-end)

1. **Design a parking lot** with a pluggable spot-assignment strategy (nearest-to-gate) and Observer notifications when a floor fills up. Add concurrency safety.
2. **Design an elevator controller** for a building with N elevators. Implement a scheduling strategy and handle simultaneous requests.
3. **Design BookMyShow**: model shows/seats and implement seat holding with a timeout so two users can't book the same seat.
4. **Design a logging framework** with levels (DEBUG/INFO/ERROR), multiple appenders (console + file), and Chain of Responsibility so a message only goes to handlers at or above its level.
5. **Design an LRU cache** with O(1) get/put, then make the eviction policy swappable (LRU vs LFU) using Strategy.
6. **Design a rate limiter** supporting both token-bucket and sliding-window algorithms behind one interface.
7. **Design a chess game**: model pieces with their own move-validation, the board, and turn/checkmate state.

For each: (a) clarify requirements, (b) list entities + relationships, (c) name the patterns and *why*, (d) code the core, (e) list extensions and concurrency concerns.

---

## 12. The 60-second summary

> "My LLD process is: clarify requirements and state assumptions, extract entities from the nouns and relationships between them, define the behaviors from the verbs, then apply patterns at real variation points — Strategy for interchangeable algorithms like pricing, Factory to hide object creation, State for mode-dependent behavior, Observer for notifications, Singleton for single-instance resources. I keep classes to a single responsibility, program to interfaces and inject dependencies (SOLID), use enums over magic values, and guard shared resources against races with locks or optimistic concurrency. I'd rather deliver a clean, extensible core and name the extensions than rush a tangled full solution."
