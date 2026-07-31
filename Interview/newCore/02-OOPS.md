# OOPS (Object-Oriented Programming) — Interview Prep Guide

> Goal: Explain the four pillars *with code and a real-world reason*, then defend design choices (inheritance vs composition, interface vs abstract class) under follow-up pressure. Examples use Java-style syntax (most common in interviews); the concepts are language-agnostic.

---

## 0. How OOPS gets tested

- **Definitions with examples:** "What is polymorphism? Give a real example." (Never define abstractly — always attach a concrete case.)
- **Compare/contrast:** overloading vs overriding, abstract class vs interface, composition vs inheritance, `==` vs `equals`.
- **Design a class hierarchy:** "Model a payment system / a set of shapes / a library." This bridges into LLD (see the LLD doc).
- **Gotchas:** "Can you override a static method?" "What is the diamond problem?" "Why favor composition over inheritance?"

---

## 1. Classes and objects — the foundation

- A **class** is a blueprint. An **object** is an instance built from that blueprint.
- Analogy: `class House` is the architectural plan; each actual house you build is an object with its own address, color, occupants.

```java
class Car {
    String brand;        // field / attribute / state
    int speed;

    void accelerate() {  // method / behavior
        speed += 10;
    }
}

Car myCar = new Car();   // object (instance)
myCar.brand = "Toyota";
myCar.accelerate();
```

**State + behavior bundled together** is the whole idea: an object holds data (fields) *and* the operations on that data (methods).

---

## 2. The Four Pillars

### Pillar 1 — Encapsulation
**What:** Bundle data + methods together, and *hide internal state* behind a controlled interface (getters/setters). Fields are `private`; access is mediated.

**Why:** You can change the internal representation without breaking callers, and you can enforce invariants (validation).

```java
class BankAccount {
    private double balance;   // hidden — no one pokes it directly

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Must be positive");
        balance += amount;
    }
    public double getBalance() { return balance; }
    // No public setBalance() — you can't just set balance to a million.
}
```
**Real-world reason to say out loud:** "If `balance` were public, any code could set it negative and break the invariant that a balance can't drop below zero via deposit. Encapsulation lets the class *guarantee* its own rules."

### Pillar 2 — Abstraction
**What:** Expose *what* an object does, hide *how*. You interact with a simplified interface and ignore the messy implementation.

**Why:** Reduces complexity for the user of a class; lets implementation change freely.

**Analogy:** Driving a car — you use the steering wheel and pedals (the interface). You don't need to know how fuel injection works (the implementation). A `List` interface says "you can `add()` and `get()`"; whether it's an ArrayList or LinkedList underneath is hidden.

```java
abstract class PaymentMethod {
    abstract void pay(double amount);   // WHAT — every payment can pay
}
class CreditCard extends PaymentMethod {
    void pay(double amount) { /* HOW: talk to card network */ }
}
class UpiPayment extends PaymentMethod {
    void pay(double amount) { /* HOW: talk to UPI gateway */ }
}
```
**Encapsulation vs Abstraction (the classic follow-up):** Encapsulation is about *hiding data* (implementation-level, achieved with access modifiers). Abstraction is about *hiding complexity/design* (design-level, achieved with abstract classes and interfaces). Encapsulation is "how you hide"; abstraction is "what you hide."

### Pillar 3 — Inheritance
**What:** A class (child/subclass) acquires fields and methods of another (parent/superclass). Models an **"is-a"** relationship.

**Why:** Code reuse and establishing type hierarchies.

```java
class Vehicle {
    int wheels;
    void start() { System.out.println("Starting..."); }
}
class Motorcycle extends Vehicle {   // Motorcycle IS-A Vehicle
    Motorcycle() { wheels = 2; }
    void wheelie() { System.out.println("Whee!"); }
}
```
**Types:** single, multilevel (`C extends B extends A`), hierarchical (many children, one parent). **Multiple inheritance of classes** is banned in Java/C# because of the **diamond problem** (below); it's allowed in C++.

**The Diamond Problem:** If B and C both inherit from A and override `foo()`, and D inherits from both B and C, which `foo()` does D get? Ambiguous. Java avoids this by disallowing multiple class inheritance; you get multiple *interface* inheritance instead (and with default methods, you must explicitly resolve the conflict).

### Pillar 4 — Polymorphism
**What:** "Many forms." The same interface/call behaves differently depending on the actual object. Two kinds:

**a) Compile-time (static) polymorphism → method overloading.** Same method name, different parameter lists. Resolved at compile time.
```java
class Calculator {
    int add(int a, int b) { return a + b; }
    double add(double a, double b) { return a + b; }
    int add(int a, int b, int c) { return a + b + c; }
}
```

**b) Runtime (dynamic) polymorphism → method overriding.** A subclass provides its own version of a parent method. Resolved at runtime based on the actual object type (dynamic dispatch).
```java
class Animal { String sound() { return "..."; } }
class Dog extends Animal { String sound() { return "Woof"; } }
class Cat extends Animal { String sound() { return "Meow"; } }

Animal a = new Dog();   // reference type Animal, object type Dog
a.sound();              // "Woof" — decided at RUNTIME
```
**Why it matters (real example):** A renderer loops over `List<Shape>` and calls `shape.draw()`. Circles, squares, triangles each draw themselves. Adding a new shape needs *zero changes* to the loop. That's the payoff — code that works with the abstraction, not the concrete types.

---

## 3. Overloading vs Overriding (near-guaranteed question)

| | Overloading | Overriding |
|---|---|---|
| Relationship | Same class (or inherited) | Parent–child classes |
| Signature | Different parameters | **Identical** signature |
| Binding | Compile time (static) | Runtime (dynamic) |
| Return type | Can differ | Same or covariant |
| Purpose | Convenience / multiple ways to call | Specialize inherited behavior |

**Gotcha:** "Can you override a `static` method?" → **No.** Static methods belong to the class, not the instance, so they're resolved at compile time by reference type — this is *method hiding*, not overriding. Also can't override `final` or `private` methods.

---

## 4. Abstract class vs Interface (the other near-guaranteed question)

| | Abstract class | Interface |
|---|---|---|
| Methods | Can have both abstract *and* concrete methods | Traditionally all abstract; modern languages allow `default` methods |
| Fields | Can have instance fields with state | Only constants (`public static final`) |
| Constructor | Yes | No |
| Multiple inheritance | A class extends only **one** | A class implements **many** |
| Models | "is-a" with shared implementation | "can-do" capability / contract |
| When to use | Related classes sharing common code + state | Unrelated classes sharing a capability |

**How to choose (say this):** "Use an abstract class when subclasses share both state and behavior and there's a clear is-a hierarchy — e.g., `Employee` → `Manager`, `Engineer` sharing a `calculatePay()` skeleton. Use an interface when I want to define a *capability* that unrelated classes can implement — e.g., `Comparable`, `Serializable`, `Flyable`, which a Bird and an Airplane can both implement without being related."

```java
// Interface = capability contract
interface Flyable { void fly(); }

class Bird implements Flyable { public void fly() { /*...*/ } }
class Airplane implements Flyable { public void fly() { /*...*/ } }
// Bird and Airplane are unrelated but both "can fly"
```

---

## 5. Composition vs Inheritance — "favor composition over inheritance"

This is a maturity signal. Interviewers love it.

- **Inheritance = "is-a".** A Dog *is an* Animal.
- **Composition = "has-a".** A Car *has an* Engine.

**Why favor composition:** Inheritance creates *tight coupling* — the child depends on the parent's implementation, and deep hierarchies become fragile ("fragile base class problem": a change in the base breaks subclasses). Composition is more flexible — you assemble behavior from parts and can swap them at runtime.

**Classic cautionary example:** You have a `Stack`. Tempting to write `class Stack extends ArrayList`. Bad — now Stack inherits `add(index, element)` and `remove(index)`, which let callers violate LIFO. Better: `class Stack { private ArrayList<T> items; ... }` — compose, and expose only `push`/`pop`.

```java
// Composition: Car HAS-A Engine (swappable)
interface Engine { void run(); }
class PetrolEngine implements Engine { public void run() {} }
class ElectricEngine implements Engine { public void run() {} }

class Car {
    private Engine engine;              // has-a
    Car(Engine engine) { this.engine = engine; }  // inject the part
    void start() { engine.run(); }
}
new Car(new ElectricEngine());  // swap behavior without touching Car
```
This also demonstrates **dependency injection**.

---

## 6. SOLID Principles (bridge to LLD — asked in both rounds)

The five principles that keep OO code maintainable. Know each with a one-line example.

- **S — Single Responsibility:** A class should have one reason to change. *E.g.,* don't make `Invoice` both calculate totals *and* print itself *and* save to DB — split into `Invoice`, `InvoicePrinter`, `InvoiceRepository`.
- **O — Open/Closed:** Open for extension, closed for modification. *E.g.,* to add a new shape, add a `Shape` subclass — don't edit a giant `if (type == circle) ... else if (type == square)` switch.
- **L — Liskov Substitution:** A subclass must be usable anywhere its parent is, without breaking behavior. *Violation:* `Square extends Rectangle` where setting width also changes height — breaks code that assumes independent sides.
- **I — Interface Segregation:** Don't force classes to implement methods they don't use. *E.g.,* split a fat `Machine { print(); scan(); fax(); }` into `Printer`, `Scanner`, `Fax` so a simple printer isn't forced to implement `fax()`.
- **D — Dependency Inversion:** Depend on abstractions, not concretions. *E.g.,* `NotificationService` depends on a `MessageSender` interface, not directly on `EmailSender` — so you can inject SMS/push later.

(Full treatment with code lives in the **LLD guide**.)

---

## 7. Key language mechanics (rapid-fire)

- **`static`:** belongs to the class, not any instance. One copy shared by all objects. Used for utility methods (`Math.max`), counters, constants. Can't access instance (`this`) members directly.
- **`final`:** `final` variable = constant; `final` method = can't override; `final` class = can't extend (e.g., `String`).
- **Constructor:** special method to initialize an object; same name as class, no return type. **Constructor overloading** = multiple constructors with different params. **Constructor chaining** = one constructor calling another (`this(...)`) or the parent's (`super(...)`).
- **`this` vs `super`:** `this` = current object; `super` = parent class (call parent constructor/method).
- **Access modifiers (Java):** `private` (same class) < `default`/package-private (same package) < `protected` (package + subclasses) < `public` (everywhere).
- **Shallow vs deep copy:** Shallow copy duplicates the object but shares references to nested objects (change one, both see it). Deep copy duplicates everything recursively (fully independent).
- **`==` vs `.equals()`:** `==` compares references (same object in memory) for objects; `.equals()` compares logical equality (override it for value comparison). If you override `equals()`, you **must** override `hashCode()` too (contract: equal objects must have equal hash codes, else they break in HashMaps/HashSets).

---

## 8. Object relationships (UML vocabulary — useful in LLD too)

- **Association:** general "uses-a" relationship. A `Teacher` is associated with `Students`.
- **Aggregation:** "has-a" with *independent* lifecycles. A `Department` has `Professors`, but professors survive if the department closes. (Hollow diamond in UML.)
- **Composition:** "has-a" with *dependent* lifecycles (strong ownership). A `House` has `Rooms`; destroy the house, the rooms go too. (Filled diamond.)
- **Inheritance:** "is-a".

**Memory aid:** Composition is a stronger form of aggregation — the part *can't exist* without the whole.

---

## 9. Design Patterns (intro — full coverage in LLD guide)

Patterns are reusable solutions to common design problems. Three categories:
- **Creational** (how objects are made): Singleton, Factory, Builder, Prototype.
- **Structural** (how objects are composed): Adapter, Decorator, Facade, Proxy, Composite.
- **Behavioral** (how objects interact): Observer, Strategy, Command, Iterator, State.

Know at least **Singleton, Factory, Observer, Strategy** cold — they come up constantly. (See LLD guide §Design Patterns.)

---

## 10. Common tricky questions (with crisp answers)

1. **Can a constructor be private?** Yes — used in Singleton (prevent external instantiation) and factory methods.
2. **What is the diamond problem and how does Java solve it?** Ambiguity from multiple class inheritance; Java bans multiple class inheritance and requires explicit resolution for conflicting interface default methods.
3. **Can you override static methods?** No — that's method *hiding*, resolved by reference type at compile time.
4. **Why is `String` immutable in Java?** Security (used in class loading, network), thread-safety (shareable without locks), and enabling the string pool (caching/interning). Any "modification" creates a new String.
5. **What's the difference between an abstract method and a concrete method?** Abstract has no body (subclass must implement); concrete has an implementation.
6. **Is Java pass-by-value or pass-by-reference?** Always pass-by-value. For objects, the *value of the reference* is passed (a copy of the pointer) — so you can mutate the object but not reassign the caller's variable.
7. **What is method hiding?** When a subclass defines a static method with the same signature as a parent's static method — resolved at compile time, not overriding.
8. **What is an inner/nested class useful for?** Logically grouping a helper that's only used by the outer class, and accessing the outer class's private members (e.g., an iterator inside a collection).
9. **What does `super()` do if you don't write it?** The compiler inserts an implicit no-arg `super()` as the first line of every constructor.
10. **When would inheritance be the wrong choice?** When the relationship is really "has-a" not "is-a", when you only want code reuse (use composition), or when the base class isn't designed for extension (fragile base class).

---

## 11. Worked design example — model a ride-hailing system's users

Shows the pillars + relationships together.

```java
// Abstraction + Inheritance: a common contract for people in the system
abstract class User {
    private String id;               // Encapsulation: private state
    private String name;
    User(String id, String name) { this.id = id; this.name = name; }
    public String getName() { return name; }
    abstract double getRating();     // each user type computes rating differently
}

class Rider extends User {           // Rider IS-A User
    private List<Trip> trips;
    Rider(String id, String name) { super(id, name); }
    double getRating() { /* avg of driver-given ratings */ return 4.8; }
}

class Driver extends User {          // Driver IS-A User
    private Vehicle vehicle;         // Composition: Driver HAS-A Vehicle
    Driver(String id, String name, Vehicle v) { super(id, name); this.vehicle = v; }
    double getRating() { /* avg of rider-given ratings */ return 4.9; }
}

// Polymorphism: process any user uniformly
void printRating(User u) {
    System.out.println(u.getName() + ": " + u.getRating());  // dispatches to Rider or Driver
}
```
Talking points: `User` is abstract (you never instantiate a generic user). `getRating()` demonstrates runtime polymorphism. `Driver has-a Vehicle` is composition. Fields are private → encapsulation.

---

## 12. Practice questions

**Explain-with-example:**
1. Give a real-world example of each of the four pillars from an app you've used.
2. Explain the Liskov Substitution Principle and show a violation in code.
3. Why does overriding `equals()` require overriding `hashCode()`? What breaks if you don't?

**Design:**
4. Model a `Shape` hierarchy (circle, rectangle, triangle) supporting `area()` and `draw()`. Which pillar makes adding a new shape cheap?
5. Design a notification system that can send Email, SMS, and Push — using composition and an interface, not a big inheritance tree.
6. Refactor a `God class` `OrderManager` that validates, prices, saves, and emails an order, applying the Single Responsibility Principle.

**Gotchas:**
7. Predict the output: a `Parent p = new Child();` calling an overridden method and an overloaded method — which version runs?
8. Explain the diamond problem with a code sketch and how you'd resolve it with interfaces + default methods.
9. When is composition strictly better than inheritance? Give a concrete refactor.

---

## 13. The 60-second summary

> "OOP organizes code around objects that bundle state and behavior. The four pillars: **encapsulation** hides internal state behind a controlled interface so classes enforce their own invariants; **abstraction** exposes what an object does and hides how; **inheritance** models is-a relationships for reuse; **polymorphism** lets one interface take many forms so code depends on abstractions, not concrete types. I favor composition over inheritance to avoid tight coupling and fragile hierarchies, I choose interfaces for capabilities and abstract classes for shared implementation, and I lean on SOLID — especially single responsibility and dependency inversion — to keep designs flexible and testable."
